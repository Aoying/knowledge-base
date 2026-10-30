---
知识点: Kafka 生产者可靠性
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-31
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

## 小技巧与冷知识

- 【冷知识】Kafka 3.0 起（KIP-679），新生产者的默认值已经是“安全组合”：`enable.idempotence=true`、`acks=all`、`retries=Integer.MAX_VALUE`、`max.in.flight.requests.per.connection=5`。原文第 2、3 节逐项配的那些参数，在 4.x 上很多只是把默认值写明。但显式写仍有价值：一旦配了冲突参数（比如 `acks=1`）而又没显式开启幂等，幂等会被静默关掉，不会报错提醒你。
- 【小技巧】调超时要成对调：官方文档要求 `delivery.timeout.ms` 应当大于等于 `request.timeout.ms` 与 `linger.ms` 之和（默认值分别是 120000、30000、0）。只调大 `delivery.timeout.ms` 而不动 `request.timeout.ms`，单次请求照样 30 秒就超时，重试预算等于没加够；反过来 `request.timeout.ms` 接近 `delivery.timeout.ms` 时，一次超时就几乎耗尽整条消息的重试预算。
- 【冷知识】Kafka 4.0 把 `linger.ms` 的默认值从 0 改成了 5：发送前多等 5 毫秒攒批，批次更大、请求更少，多数场景下延迟不升反降。原文没提这个参数，但它是“异步攒批”真正的节拍器；把 `linger.ms` 设得很大（如几百毫秒）之前，先确认业务能不能接受这段固定等待。
- 【小技巧】回调里拿到失败，不等于消息没写入：`delivery.timeout.ms` 超时、确认在路上丢了，消息都可能已经落盘，回调给你的其实是“结果未知”，而不是“肯定没发”。所以失败兜底不能无脑重发整条业务消息，要先靠业务幂等键或去重表判断这条是不是已经写进去了，否则超时重试本身就会造重复。
- 【冷知识】`acks=all` 里的“all”只是 ISR 全体，不是全部副本：掉出 ISR 的副本不参与确认。ISR 收缩时 `acks=all` 的实际耐久性会下降，这正是 `min.insync.replicas` 必须 ≥2 的原因——它卡的是“至少几个同步中的副本确认”，而不是副本总数。
- 【小技巧】幂等生产者防得住客户端重试，防不住应用层重发：broker 去重只认 PID + 分区序列号，客户端内部重试携带同一序列号才会被识别；业务代码自己重新 `send()` 一次是新的序列号，broker 会当成两条不同消息照单全收。回调失败后想重发，走失败表 + 幂等键的业务兜底，别在应用层再调一次 `send()`。
- 【冷知识】事务生产者重启有“围栏”机制：同一 `transactional.id` 重启后，新实例的 epoch 会加 1，旧实例如果没死透、还在写，事务协调器会用 `ProducerFencedException` 把它挡下来，避免两个实例同时写。这就是 `transactional.id` 必须稳定的深层原因——它不只是个名字，还是区分“新旧 incarnation”的身份。
- 【小技巧】退出前别只调无参 `close()`：无参关闭会一直等缓冲区排空，broker 挂了或网络断了时可能长时间卡住、关不掉进程。生产代码用带超时的 `close(Duration)`，超时后还没发完的消息按失败处理、记失败表兜底，而不是让关机流程无限等下去。
- 【冷知识】`read_committed` 消费者跳过已回滚的事务时，位点照样推进：abort 的消息仍占 offset，消费者直接跳过、offset 继续前进。所以“消费位点”和“可见消息数”对不上是正常的，排查“丢消息”时先看是不是回滚事务留下的空洞，别一上来就怀疑生产者丢了数据。
- 【小技巧】`acks=0` 时 `retries` 根本不生效：官方文档写明，`acks=0` 下生产者收不到任何失败信号、不知道消息有没有到 broker，重试无从谈起，回调里拿到的 offset 恒为 -1。`acks=0` 不是“快一点的可靠发送”，而是彻底放弃确认，别把它和重试、幂等搭配着配。
- 【小技巧】无 key 时默认分区器是粘性（sticky）的：同一批次的消息粘在同一个分区里攒批，批次之间再轮换分区。原文说“不需要保序就别设 key，让消息打散换吞吐”，sticky 分区器正是这个行为的实现——既攒批省请求，又不会把所有无 key 消息堆到同一个分区。
- 【冷知识】`max.block.ms` 默认 60 秒，是 `send()` 的阻塞上限：缓冲区满或元数据不可用时，`send()` 最多阻塞 60 秒，之后抛异常。原文第 6 节说缓冲区打满是背压信号，这个 60 秒就是背压变成显性异常的时刻；调大 `buffer.memory` 之前，先确认这 60 秒里 broker 与网络是不是真的有问题。

## 面试与自测

1. `acks=all` 单独开为什么还不够？“不丢”的三件套是什么？
2. 幂等生产者解决什么重复、管不了什么重复？开启后对配置有什么硬约束？
3. Kafka 事务消息和 RocketMQ 事务消息解决的是同一个问题吗？消费者侧要配什么才看不到回滚的消息？

### 参考答案

1. 还不够，必须和 broker 侧参数配套：`min.insync.replicas`（≥2），ISR 少于这个数时直接拒绝写入，宁可失败也不假成功；再加 `unclean.leader.election.enable=false`，非 ISR 副本不许当选 leader；三件套齐了才叫“不丢”。代价是延迟，等多个副本确认必然比 `acks=1` 慢。
2. 解决的是单生产者、单会话内的重试重复：broker 给每个生产者分配 PID，并对每个分区维护发送序列号，重试携带同一序列号，broker 识别出重复批次直接丢弃。生产者重启换了新 PID，序列号体系重来，跨会话的重复它管不了，只能靠事务或业务幂等兜底。硬约束：开启后要求 `acks=all`、`retries>0`、`max.in.flight.requests.per.connection ≤ 5`，配置不满足直接报错。
3. 不是同一个问题：Kafka 事务解决跨分区原子写 + 消费位点与产出消息的原子性（consume-transform-produce 链路的 exactly-once 就靠它）；RocketMQ 是 half message + 事务回查，解决“本地事务与发消息一致”，别混着选型。消费者侧要配 `isolation.level=read_committed` 才看不到已回滚的事务消息，默认 `read_uncommitted` 等于没隔离。

## 8. 关联知识

- 总览与选型：[Kafka 消息队列](消息队列/Kafka-消息队列.md)
- 消费端配套：[Kafka 消费者组与 Rebalance](消息队列/Kafka-消费者组与Rebalance.md)
- 三档投递语义的边界与选型清单：[消息可靠性](消息队列/消息可靠性.md)
