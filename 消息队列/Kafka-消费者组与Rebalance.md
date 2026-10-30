---
知识点: Kafka 消费者组与 Rebalance
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】给 group.id 加上环境和用途后缀，例如 orders-pay-notify-prod，而不是多个不同处理逻辑共用同一个组名。因为位点是按组名保存的，共用组名会让两套逻辑抢同一批分区、互相推进位点，轻则丢处理、重则重复，排查时很难分清是谁提交了位点。

【小技巧】下游变慢时优先用 consumer.pause() 暂停分区，而不是在 poll 循环里长时间 sleep。暂停后 poll 仍在正常调用，不会触发 max.poll.interval.ms 超时被踢出组；下游恢复后再用 resume() 继续拉取，这比调大超时参数更安全，也不会掩盖真正的慢处理问题。

【小技巧】查 lag 不要只看组总和，要用 kafka-consumer-groups --describe 看每个分区的 CURRENT-OFFSET、LOG-END-OFFSET 和 LAG。总 lag 正常也可能有个别分区长期堆积，这通常意味着该分区对应的 key 分布不均或某条消息反复慢处理，只看总量会漏掉这种倾斜。

【小技巧】max.poll.records 只限制条数，不限制字节数，单条大消息仍可能让一批处理很重。如果消息体偏大，要同时收紧 max.partition.fetch.bytes 和 fetch.max.bytes，用字节数给单批上界，再用 max.poll.records 控制条数，两者配合才能真正控住单批耗时。

【小技巧】需要重放某段时间的数据时，用 offsetsForTimes 按时间戳查到位点再 seek，不要为了重放就新建一个 group.id。新组名会从头或从最新开始消费整组分区，范围不好控；按分区 seek 只动目标分区和时间段，重放完还能回到原来的位点继续正常消费。

【小技巧】把 commitAsync 放在正常循环里、commitSync 放在关闭和分区撤销时，是常见的组合。异步提交不阻塞消费主循环，失败也不致命；但进程退出或分区交接是最后一次提交机会，这时必须用同步提交确认成功，否则刚处理完的位点可能没落盘，下次就重复消费。

【冷知识】消费位点存在内部主题 __consumer_offsets 里，它的默认分区数是 50，分区键由 group.id、topic 和分区号组成。组协调器（coordinator）就是按 group.id 的哈希落在 __consumer_offsets 的某个分区上、由拥有该分区的 broker 担任，所以换组名等于换了协调器和位点记录，接不上旧进度。

【冷知识】每次 rebalance 都会让消费组的 generation id 加一，旧成员再用旧 generation 发心跳或提交时会被协调器拒绝。这就是 rebalance 的围栏机制：防止已被踢出组、但还没发现自己被踢的消费者继续提交位点，避免它把新成员的进度覆盖掉。

【冷知识】心跳线程和 poll 线程是分开的，这一点来自 KIP-62，从 Kafka 0.10.1 开始心跳就由后台线程独立发送。所以“心跳正常”只能证明进程和网络还活着，证明不了业务线程没卡住，这也是原文第 4 节说排查时别只看“进程还在”的历史原因。

【冷知识】协作式 rebalance 来自 KIP-429，在 Kafka 2.4 引入 CooperativeStickyAssignor；静态成员来自 KIP-345，在 2.3 引入。两者出现的时间比很多人以为的早，但因为要全组成员客户端都支持协作协议才能生效，旧客户端混用时会退回老行为，所以升级时要先确认全组客户端版本一致。

【冷知识】Kafka 4.0 的新组协议（KIP-848）把分区分配从消费组 leader 消费者手里收回，改由 broker 侧的协调器直接做。老协议下每次 rebalance 都要先选出一个 leader 消费者来算分配表，新协议省掉了这一步，也让增量交接更彻底，这是 4.x 打断更小的一个结构性原因。

## 面试与自测

1. 四种分区分配策略中当前推荐哪种？它比老的 Eager 协议好在哪里？
2. 心跳正常，但消费者仍被踢出组，可能是什么原因？排查时要注意什么？
3. 为什么“先处理、后提交”得到 at-least-once，而反过来会丢？分区被撤销时还要做什么？

### 参考答案

1. 推荐 CooperativeSticky：粘性 + 协作式（增量）rebalance，分两步交接，不必全组停顿。老的 Eager 协议 rebalance 是全组先全部撤销、再重新分配，期间整组停消费；4.x 的新组协议（KIP-848）进一步把分配移到 broker 侧做增量协调，打断更小。
2. 心跳由后台线程发送，与 poll 是两条线：进程没死但业务线程卡在慢调用上，心跳正常、poll 间隔超过 `max.poll.interval.ms`（默认 5 分钟）照样被判定“活着但卡死”而踢出组。排查时别只看“进程还在”，要看单批处理耗时；批量消费的重试总耗时也算单批耗时，退避拉太长同样会触发 rebalance。
3. 先处理、后提交得到 at-least-once：崩溃点之后的重复靠业务幂等消化；反过来先提交后处理是 at-most-once，消息还没处理完位点已提交就丢了。另外要在 `ConsumerRebalanceListener` 的 `onPartitionsRevoked` 里把已处理完的位点同步提交掉再交出分区，能缩小重复窗口。

## 8. 关联知识

- 总览与选型：[Kafka 消息队列](消息队列/Kafka-消息队列.md)
- 生产端配套：[Kafka 生产者可靠性](消息队列/Kafka-生产者可靠性.md)
- 投递语义与消费端幂等模式：[消息可靠性](消息队列/消息可靠性.md)
