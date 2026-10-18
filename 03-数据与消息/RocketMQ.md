# RocketMQ

> 每晚 20:30 自动补充完善。版本相关内容以官方资料为准（2026-10 核验）。

## 一句话定位

阿里开源的分布式消息中间件，强项是**业务消息**：事务消息、延迟消息、顺序消息开箱即用；Kafka 强项是**大数据管道**：极致吞吐 + 流处理生态。选型看场景，不看熟悉度。

## 核心概念

- **NameServer**：轻量路由注册中心，无状态、各节点互不通信；Broker 每 30s 心跳注册，Producer/Consumer 从它拿路由。
- **Broker**：存消息。`CommitLog`（所有 topic 顺序混合写，1GB/个）+ `ConsumeQueue`（按 topic/queue 建索引，存偏移量/大小/Tag 哈希）→ 单机撑 5 万队列不掉性能。
- **Topic / Queue**：Topic 拆多个 Queue（默认 4），Queue 是实际存储和消费单元。
- **Producer Group / Consumer Group**：同一 Group 的 Consumer 均摊 Queue（集群消费）或全量广播（广播消费）。

## 5.x 架构变化

5.0 起三层：SDK 层（gRPC 多语言轻量 SDK）→ **Proxy 层**（新增，处理连接/路由/续期，存算分离）→ 存储层（NameServer + Broker）。
**Pop 消费模式**：无状态消费，单条消息级别的消费/重试/删除 API。
**任意精度定时消息**：5.x 基于时间轮实现，4.x 只有 18 个固定延迟等级（1s~2h）。

## 生产者要点

```java
DefaultMQProducer producer = new DefaultMQProducer("my-group");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();
Message msg = new Message("TopicTest", "TagA", "key-1", "body".getBytes());
producer.send(msg); // 同步发送；sendOneway 吞吐最高但无回执
producer.shutdown();
```

- 重试：`setRetryTimesWhenSendFailed(3)`，同步发送失败内部重试。
- 发送超时默认 3s，大消息调 `setSendMsgTimeout`。

## 消费者要点

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("my-group");
consumer.setNamesrvAddr("127.0.0.1:9876");
consumer.subscribe("TopicTest", "TagA || TagB"); // Tag 过滤在 Broker 端做
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, ctx) -> {
    // 业务处理；返回 CONSUME_SUCCESS / RECONSUME_LATER
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
consumer.start();
```

- 并发消费 `MessageListenerConcurrently` vs 顺序消费 `MessageListenerOrderly`（同一 queue 单线程）。
- 消费位移存在 Broker 端，重平衡后自动续消费。

## 事务消息（半消息二阶段）

1. Producer 发 Half 消息（暂不投递）；2. 执行本地事务；3. 发 Commit/Rollback。
Broker 长时间没收到确认会**回查**（默认 15 次）→ 保证最终一致。适合"本地 DB 事务 + 发消息"场景，比本地消息表方案开发成本低。

## 延迟 / 顺序消息

- 延迟：4.x 18 个固定等级；5.x 任意时间精度。典型：下单 30 分钟未支付自动取消。
- 顺序：全局顺序（单 queue 单 consumer）或分区顺序（`MessageQueueSelector` 按业务键路由到同一 queue + `MessageListenerOrderly`）。

## 与 Kafka 选型对比

| | RocketMQ | Kafka |
|---|---|---|
| 事务消息 | 原生半消息+回查 | 仅发送端事务（EOS 语义） |
| 延迟消息 | 原生支持 | 无，需外部实现 |
| Tag 过滤 | Broker 端支持 | 不支持 |
| 死信队列 | 原生 | 无 |
| 吞吐 | 约 10 万/s | 约 17 万/s（sendfile 零拷贝） |
| 适用 | 订单/支付/金融业务链路 | 日志采集/实时计算/数据管道 |

一句话标准：听到 Spark/Flink 选 Kafka，其他业务场景优先 RocketMQ。

## 5 个常见坑

1. **NameServer 地址配错**：报 `No route info`，先检查 `setNamesrvAddr` 和 9876 端口连通性。
2. **Tag 写 `*` 全量消费**：`subscribe(topic, "*")` 会拉全量，生产环境按需写 Tag 表达式。
3. **顺序消费里抛异常**：`MessageListenerOrderly` 中异常会导致该 queue 阻塞，务必 try-catch 并记录。
4. **消息体过大**：默认上限 4MB，大消息拆分或走对象存储 + 传引用。
5. **消费堆积不告警**：Prometheus + Grafana 监控 queue 深度和消费延迟，设堆积阈值告警——生产保命项。

## 疑难杂症

### 1. 消费位点重置与消息重放
- **现象**：上线了 bug 版本，消费了一批脏数据，想重放。
- **原因**：消费位点（offset）是按消费组记录的，重置位点即可从指定时间重新消费。
- **解决**：控制台找到消费组 → "重置消费位点" → 选时间点；注意这是危险操作，先停掉消费者再重置，避免重复/丢失。

### 2. 事务消息回查超时
- **现象**：事务消息发出去，broker 反复回查，本地事务状态查不到。
- **原因**：`checkLocalTransaction` 实现太慢或抛异常，超过回查超时时间。
- **解决**：回查方法只做轻量查询（查本地事务表），别放重逻辑；回查超时参数按业务调大；本地事务表必须落库，不能放内存。

### 3. Broker 磁盘满拒绝写入
- **现象**：生产者报 `SYSTEM_BUSY` / `BROKER_DISK_FULL`，消息发不出去。
- **原因**：CommitLog 占满磁盘，broker 进入保护状态拒绝写入。
- **解决**：紧急扩容或清理过期文件；长期看调小文件保留时长（`fileReservedTime`）；监控磁盘水位告警，别等满了才发现。

## 顺序消息实战

```java
// 生产者：同一订单的消息用同一个 sharding key → 同一队列 → 有序
SendResult r = producer.send(new Message("orders", ("order-" + orderId).getBytes()),
    (queues, msg, arg) -> queues.get(((String) arg).hashCode() % queues.size()),
    orderId);

// 消费者：MessageListenerOrderly（单队列单线程消费，天然有序）
consumer.registerMessageListener((MessageListenerOrderly) (msgs, ctx) -> {
    // 处理 msgs
    return ConsumeOrderlyStatus.SUCCESS;
});
```

- 顺序消息 = "分区有序"：全局有序靠单队列，吞吐低；一般用"业务 key 分区有序"就够了。
- 有序消费时一个消息卡住会堵住整个队列，消费逻辑必须幂等 + 快速失败。
- 和 Kafka 对比：Kafka 用 key 分区天然有序，RocketMQ 要显式选队列，两者思想一样。
