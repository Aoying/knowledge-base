---
知识点: Kafka 消费者组与 Rebalance
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-23
关联:
  - Kafka-消息队列
  - Kafka-生产者可靠性
  - 消息可靠性
标签:
  - 消息队列
  - 原理
  - 实践
---

# Kafka 消费者组与 Rebalance

> 从 [Kafka 消息队列](消息队列/Kafka-消息队列.md) 拆分。消费端可靠性的核心就两件事：分区怎么分（决定并行度和 rebalance 行为）、位点什么时候提交（决定丢还是重复）。

## 1. 核心概念

- **Consumer Group**：一组消费者以组名 `group.id` 为身份共同消费一个 topic；组名就是身份，改了组名等于换人，历史位移对不上。
- **分配单位是分区**：一个 partition 同一时刻只被组内一个消费者消费，所以组内有效消费者数 ≤ 分区数，多出来的闲置。
- **位点（offset）存在服务端**：消费进度记在内部 topic `__consumer_offsets` 里，不是客户端本地文件；换机器消费能接上进度，改组名则接不上。
- **Coordinator**：broker 侧负责组成员管理与分区分配的角色，消费者靠心跳向它报活。

## 2. 分区分配策略

由 `partition.assignment.strategy` 配置，常用的四种：

| 策略 | 分配方式 | 特点 |
|---|---|---|
| Range | 按 topic 逐个把分区连续分段 | 同一 topic 内连续，但多 topic 时容易倾斜 |
| RoundRobin | 所有分区轮流发 | 均匀，但每次 rebalance 变动大 |
| Sticky | 尽量粘住原分配 | rebalance 时少搬动，减少重复消费窗口 |
| CooperativeSticky | 粘性 + 协作式（增量）rebalance | 分两步交接，不必全组停顿，是当前推荐 |

- 老的 Eager 协议 rebalance 是「全组先全部撤销、再重新分配」，期间整组停消费；Cooperative（协作式）分两轮增量交接，未变动的分区可继续消费。4.x 的新组协议（KIP-848）进一步把分配移到 broker 侧做增量协调，打断更小（兼容细节以官方文档为准）。

## 3. Rebalance：触发与危害

**触发条件**：

- 组成员加入（新消费者启动、扩容）或离开（下线、崩溃）。
- 订阅的 topic 集合变化，或 topic 分区数变化。
- 心跳超时：`session.timeout.ms` 内没收到心跳，coordinator 判定成员死亡。
- 两次 poll 间隔超时：超过 `max.poll.interval.ms` 还没 poll，判定「活着但卡死」，同样踢出组。

**危害**：

- rebalance 期间对应分区暂停消费，频繁触发直接拉高 lag。
- 分区在交接瞬间可能被两个消费者先后处理同一批消息，产生**重复消费窗口**——所以消费端必须幂等，这是设计前提不是可选项。
- 风暴模式：消费太慢 → poll 超时被踢 → 重新入组 → 又触发 rebalance → 更慢，正反馈循环（见疑难杂症）。

**缓解**：用 Sticky / CooperativeSticky 减少搬动面；静态成员（`group.instance.id`）让重启后在 `session.timeout.ms` 内回来的实例认领回原分区，滚动发布时显著减少 rebalance；单批别贪大（见第 4 节参数关系）。

## 4. 心跳与 poll 间隔

| 参数 | 作用 | 经验关系 |
|---|---|---|
| `session.timeout.ms` | 心跳超时判死 | 与 GC 停顿、网络抖动一起评估，别设太小 |
| `heartbeat.interval.ms` | 心跳发送间隔 | 通常 ≤ `session.timeout.ms` 的 1/3 |
| `max.poll.interval.ms` | 两次 poll 最大间隔（默认 5 分钟） | 必须 > 单批最长处理耗时 |
| `max.poll.records` | 单次 poll 最多返回条数 | 用它控制单批耗时，而不是硬扛大批量 |

- 心跳由后台线程发送，与 poll 是两条线：进程没死但业务线程卡在慢调用上，心跳正常、poll 超时，照样被踢——排查时别只看「进程还在」。
- 批量消费的重试总耗时也算单批耗时：退避拉太长，中途 poll 超时触发 rebalance，重试白做。

## 5. 位点提交

- **自动提交**（`enable.auto.commit=true`）：按间隔替你提交，省事但语义含糊——可能消息还没处理完位点已提交（丢），也可能处理完还没提交就崩了（重复）。
- **手动提交是重要链路的默认**：整批处理成功后 `commitSync()`；`commitAsync()` 不阻塞但失败不重试，常与下一次提交或关闭时的同步提交兜底组合使用。
- **先处理、后提交**得到 at-least-once：崩溃点之后的重复靠业务幂等消化。反过来先提交后处理，是 at-most-once，会丢。
- **revoke 时先提交**：实现 `ConsumerRebalanceListener`，在 `onPartitionsRevoked` 里把已处理完的位点同步提交掉，再交出分区，能缩小重复窗口。
- Spring Kafka 对应做法：关自动提交（`enable-auto-commit: false`），`@KafkaListener` 批量消费 + `Acknowledgment.acknowledge()`，业务成功才 ack。

```java
@KafkaListener(topics = "orders", groupId = "g1")
public void onBatch(List<ConsumerRecord<String, String>> rs, Acknowledgment ack) {
    try { process(rs); ack.acknowledge(); }
    catch (Exception e) { /* 别 ack，下次重投；多次失败的走死信 */ }
}
```

对应配置：`spring.kafka.consumer.enable-auto-commit: false`、`max-poll-records: 50`、`properties.max.poll.interval.ms: 300000`（按单批耗时调整）。

## 6. 消费失败与死信队列（DLT）

处理失败的消息不能无限重试堵住分区，生产标准做法是「有限重试 → 死信队列」：

```java
@Bean
public DefaultErrorHandler kafkaErrorHandler(KafkaTemplate<String, String> kafkaTemplate) {
    // 重试耗尽后发往 <原topic>.DLT，header 带原始 topic/partition/offset/异常信息
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
    ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(4);
    backOff.setInitialInterval(1_000L);   // 1s→2s→4s→8s（封顶 10s），共 5 次尝试
    backOff.setMultiplier(2.0);
    backOff.setMaxInterval(10_000L);
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
    handler.addNotRetryableExceptions(ValidationException.class); // 校验类错误直接进 DLT
    return handler;
}
```

- `@DltHandler` 接收死信后必须**告警 + 可重放**，只记日志等于扔进黑洞；DLT 保留期设得比原 topic 长，配「读 DLT → 重发原 topic」的重放脚本。
- DLT topic 分区数至少与原 topic 一致（默认按原分区号投递），否则改投递目标解析做取模映射；批量消费时失败的那条进 DLT、其余 offset 照常提交，别让毒消息卡住整批。

## 7. 疑难杂症

### Rebalance 风暴

- **现象**：消费组反复进出、lag 锯齿状波动，日志里频繁出现 rebalance。
- **原因**：单批处理耗时超过 `max.poll.interval.ms` 被踢出组；或长 GC、慢下游让 poll 间隔失控。
- **解决**：调小 `max.poll.records` 或调大 `max.poll.interval.ms`；慢处理改异步/拆批；滚动发布场景加静态成员。

### 消费者 lag 持续增长的排查链路

- **现象**：lag 只增不减，消费组像没在干活。
- **排查顺序**：消费者挂了/陷在 rebalance 风暴 → 单条消费耗时（下游慢）→ 分区数 < 消费线程数（扩线程没用）→ broker 磁盘/网络瓶颈。
- **解决**：90% 的 lag 是下游慢；先看耗时，再确认分区数 ≥ 线程数；临时扩分区 + 消费者顶过去再查根因。

## 8. 关联知识

- 总览与选型：[Kafka 消息队列](消息队列/Kafka-消息队列.md)
- 生产端配套：[Kafka 生产者可靠性](消息队列/Kafka-生产者可靠性.md)
- 投递语义与消费端幂等模式：[消息可靠性](消息队列/消息可靠性.md)
