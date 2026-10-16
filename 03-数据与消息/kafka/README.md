# Kafka

> 每晚 20:30 自动补充完善。时效性以 Apache Kafka 官方为准（2026 年主线 4.x，已彻底去掉 ZooKeeper，全面 KRaft）。

## 核心概念一句话

- **Topic**：消息的逻辑分类；**Partition**：topic 的分片，并行和顺序性的基本单位。
- **Offset**：partition 内的消息序号，消费进度就是 offset。
- **Consumer Group**：一组消费者共同消费一个 topic，一个 partition 同一时刻只被组内一个消费者消费。
- **Broker**：Kafka 服务节点；**KRaft**：4.x 的元数据共识机制（替代 ZooKeeper）。

## 生产者关键参数

```properties
acks=all                # leader+所有ISR副本都写成功才算成功，最安全
enable.idempotence=true # 幂等生产者，防重试导致重复
retries=Integer.MAX_VALUE
# 事务（exactly-once 语义需要）：transactional.id + initTransactions
```

- `acks=1` 够快但可能丢；`acks=0` 别在生产用。
- **顺序性**：同一个 key 的消息进同一个 partition 才有序；跨 partition 无序。

## 消费者关键行为

- **位移提交**：`enable.auto.commit=true` 省事但可能丢/重复；重要消费手动 `commitSync`。
- **Rebalance**：组成员变化触发分区重分配，期间暂停消费；用 `ConsumerRebalanceListener` 在 revole 时先提交位移。
- **消费能力上限**：一个 group 里有效消费者数 ≤ partition 数，多出来的闲置。

## 4.x 值得知道的变化

- **KRaft**：不再需要 ZooKeeper，部署和运维简单一截；新集群直接用 KRaft 模式。
- **Share Groups（4.2 起生产可用）**：队列语义——多消费者可协作消费同一 partition，带单条 ack；适合"既用 Kafka 又要队列语义"的场景，不为此单独搭 RabbitMQ。

## 常见坑

1. **消息丢失**：生产者 `acks=1` + leader 挂了；消费者自动提交 + 处理失败。重要链路：生产者 `acks=all` + 幂等，消费者手动提交。
2. **消息重复**：at-least-once 是常态，消费端做**幂等**（业务主键去重表）。
3. **Rebalance 风暴**：消费太慢触发 `max.poll.interval.ms` 超时被踢出组 → 调大该值或加快单次 poll 处理。
4. **分区数拍脑袋**：分区数决定最大并行度，定少了以后加分区会导致 key 到分区的映射变化（顺序性受影响）。
5. **__consumer_offsets  itself 膨胀**：正常现象，别手贱去删。

## 待补充

- Spring Kafka（KafkaTemplate / @KafkaListener）实战片段
- RocketMQ vs Kafka 选型对比（见 rocketmq/ 方向）
- 本地 Docker 单机 KRaft 起一个 broker 的 compose
