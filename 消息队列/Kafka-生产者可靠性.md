---
知识点: Kafka 生产者可靠性
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-23
关联:
  - Kafka-消息队列
  - Kafka-消费者组与Rebalance
  - 消息可靠性
标签:
  - 消息队列
  - 实践
---

# Kafka 生产者可靠性

> 从 [Kafka 消息队列](消息队列/Kafka-消息队列.md) 拆分。生产端「不丢、不重、不乱序」分别由不同机制负责，先分清再配参数，别指望一个开关全包。

## 1. acks：确认到什么程度才算发成功

| `acks` | 语义 | 风险 |
|---|---|---|
| `0` | 发出即不管 | broker 没收到也认为成功，生产别用 |
| `1` | leader 落盘即确认 | leader 刚确认就宕机、follower 没同步到，丢 |
| `all`（-1） | leader + 全部 ISR 副本同步才确认 | 最稳，必须与 broker 侧参数配套 |

- `acks=all` 单独开没用：要和 broker 的 `min.insync.replicas`（≥2）配套——ISR 少于这个数时直接拒绝写入，宁可失败也不假成功。再加 broker 的 `unclean.leader.election.enable=false`（非 ISR 副本不许当选 leader），三件套齐了才叫「不丢」。
- 代价是延迟：等多个副本确认必然比 `acks=1` 慢，重要链路认这个成本，日志类链路可以放宽。

## 2. 重试与它带来的两个副作用

```properties
acks=all
retries=Integer.MAX_VALUE          # 配合下面的总超时一起用，别单独理解成"永远重试"
delivery.timeout.ms=120000         # 一条消息从发到确认的总预算，超时才真正报失败
enable.idempotence=true            # 幂等生产者（见下节），开启后 retries 等参数有约束
```

- **重复**：消息已落盘、确认在路上丢了，生产者以为失败而重试，broker 收两份——这是重试的固有副作用，靠幂等生产者消除。
- **乱序**：多批在途时先发的批次失败重试，后发的先落盘，分区内顺序就反了。`max.in.flight.requests.per.connection` 大于 1 时存在这个风险；开幂等后 Kafka 用序列号在 broker 侧重排，这个问题一并解决。
- 重试不是越久越好：`delivery.timeout.ms` 是总预算，业务发送接口的超时要比它大，否则客户端已放弃、生产者还在后台重试，行为对不齐。

## 3. 幂等生产者

- 原理：开启 `enable.idempotence=true` 后，broker 给每个生产者分配 PID，并对每个分区维护发送序列号；重试携带同一序列号，broker 识别出重复批次直接丢弃。解决的是**单生产者、单会话内**的重试重复。
- 约束：开启后要求 `acks=all`、`retries>0`、`max.in.flight.requests.per.connection ≤ 5`，配置不满足直接报错——这是好事，别为了绕报错关幂等。
- 边界：生产者重启换了新 PID，序列号体系重来，跨会话的重复它管不了；那种重复只能靠事务或业务幂等兜底。

## 4. 事务消息（概念）

```properties
transactional.id=orders-producer-1   # 稳定的事务 id，同一业务身份重启后保持不变
```

```java
producer.initTransactions();
producer.beginTransaction();
producer.send(record1);
producer.sendOffsetsToTransaction(offsets, groupMetadata); // 消费-处理-生产场景：把消费位点也纳入同一事务
producer.commitTransaction();   // 或 abortTransaction() 回滚，两阶段提交，消息对 read_committed 消费者才可见
```

- 解决的是**跨分区原子写 + 消费位点与产出消息的原子性**（consume-transform-produce 链路的 exactly-once 就靠它）。
- 消费者侧要配 `isolation.level=read_committed` 才看不到已回滚的事务消息；默认 `read_uncommitted` 等于没隔离。
- 与 RocketMQ 事务消息不是一回事：RocketMQ 是 half message + 事务回查，解决「本地事务与发消息一致」；Kafka 事务解决「多条消息 + 位点」的原子性，别混着选型。
- 成本实打实：事务协调、多一轮提交、吞吐与延迟都受影响（见疑难杂症），非必要不开。

## 5. 顺序性

- Kafka 只保证**分区内有序**：同一个 key 的消息进同一个分区才有序，跨分区无序，这是模型定义不是参数问题。
- key 的选择即保序维度：按下单用户保序就用 userId 做 key；不需要保序就别设 key，让消息打散换吞吐（key 设计细则见 [Kafka 消息队列](消息队列/Kafka-消息队列.md) 的分区 key 一节）。
- 不开幂等时，重试 + 多在途会破坏顺序（见第 2 节）；强顺序链路要么开幂等，要么把 `max.in.flight.requests.per.connection` 设为 1 认吞吐损失。

## 6. 发送回调与异常处理

```java
producer.send(new ProducerRecord<>("orders", userId, orderJson), (meta, ex) -> {
    if (ex != null) { /* 发送失败兜底：记失败表/告警，别在回调里再同步重试堵住发送线程 */ }
    else { /* meta.topic()/partition()/offset() 是这条消息的最终坐标，排查时全靠它 */ }
});
producer.close();   // 退出前必须 close/flush，没发完就关进程会丢消息
```

- 发送是异步攒批的：`send()` 返回只代表进了缓冲区；同步 `get()` 等结果只适合排障和低频场景，业务主链路用回调。
- 异常分两类看：可重试的（网络抖动、leader 选举中、ISR 变动）由内部重试消化，回调里拿到的是重试耗尽后的最终失败；不可重试的（消息过大、序列化错误、参数非法）直接失败，记下来修数据或代码，别盲目重发。
- 缓冲区打满（`buffer.memory` 不够、`max.block.ms` 超时抛异常）是背压信号：先查 broker 与网络，别只调大缓冲掩盖问题。
- Spring Kafka 的 `KafkaTemplate.send` 返回 future，异常处理与死信/失败表落点要在发送侧就设计好，别只管消费端。

## 7. 疑难杂症

### exactly-once 的代价

- **现象**：开了幂等 + 事务，吞吐明显下降、延迟升高。
- **原因**：exactly-once = 幂等生产者 + 事务 + 消费者 `read_committed` + 消费端处理，三重成本叠加。
- **解决**：不是所有业务都要 exactly-once；能接受「至少一次 + 消费端幂等」就别开事务；真要开，先压测出基准再上线。

### ISR 收缩与 unclean leader election 丢数据

- **现象**：broker 挂了一个，恢复后发现少了一段消息。
- **原因**：`unclean.leader.election.enable=true` 时，非 ISR 副本也能当选 leader，它没同步到的数据就丢了。
- **解决**：生产设 `unclean.leader.election.enable=false`，宁可短暂不可用也不丢数据；`min.insync.replicas` 至少 2，与 `acks=all` 配套。

## 8. 关联知识

- 总览与选型：[Kafka 消息队列](消息队列/Kafka-消息队列.md)
- 消费端配套：[Kafka 消费者组与 Rebalance](消息队列/Kafka-消费者组与Rebalance.md)
- 三档投递语义的边界与选型清单：[消息可靠性](消息队列/消息可靠性.md)
