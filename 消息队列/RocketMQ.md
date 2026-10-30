---
知识点: RocketMQ
层级: 进阶
适用版本: 通用
更新日期: 2026-10-31
关联:
  - 数据库知识点地图
  - Kafka-消息队列
  - MySQL-数据库
标签:
  - 消息队列
  - 实践
---

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

> 横向专题：[消息队列/消息可靠性.md](消息队列/消息可靠性.md)（三档语义边界、Kafka/RocketMQ 机制对比、消费端幂等三种模式、回查 checklist）。

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

## 延迟消息

```java
// 18 个固定等级：1s 5s 10s 30s 1m 2m ... 2h（不能自定义时长）
msg.setDelayTimeLevel(3);  // 10s 后投递
producer.send(msg);
```

- 场景：订单 30 分钟未支付关单、定时提醒。
- 只能选固定等级，`setDelayTimeLevel(4)` = 30s，想"17 分钟"就选 10m 那档或自己轮询。
- 延迟消息不支持重试太多次，消费失败进死信后延迟语义就没了，消费逻辑要幂等。

## 基础知识补强

### 核心概念与术语

- 为什么用消息队列，心智模型三件事：解耦（下单只管发消息，发短信/加积分各自订阅，下游挂了不拖垮下单）、异步（非核心流程挪出主链路，接口先返回）、削峰（秒杀流量先在队列里排队，下游按自己节奏消费）。代价：系统多一个组件、要处理重复和顺序、排查链路变长。
- 消息的 anatomy：一条消息 = Topic（发到哪）+ Tag（二级分类，消费端过滤用）+ Key（业务唯一键，查单条消息、排障定位用）+ Body（正文）+ MessageId（系统生成的全局 ID，与 Key 是两回事）。
- Tag vs Key 别混：Tag 决定「谁来消费」（Broker 端过滤），Key 决定「怎么查这条消息」（运维和对账用）；一条消息一个 Tag、多个 Key 都行。
- 消费模式两种：集群消费（默认，一个 Group 内分摊消息，每条只被一个消费者处理，做业务用它）与广播消费（组内每个消费者都收全量，只适合配置下发、缓存刷新这类「人人都要知道」的场景）；用错模式是最常见的事故源。
- 消费进度 offset：每个队列消费到哪一条，进度归消费组所有；换组 = 从头（或从配置的起点）重新消费，同一组重启则接着进度走。
- 死信队列：消费重试耗尽的消息被投递到 `%DLQ% + 消费组名` 的特殊 Topic，不再自动重投，要人工查原因后重新发送。

### 基础语法与用法

```bash
# 集群自检（mqadmin，运维基础命令）
mqadmin clusterList -n 127.0.0.1:9876            # 集群和 Broker 状态
mqadmin topicList -n 127.0.0.1:9876              # 全部 Topic
mqadmin topicStatus -n 127.0.0.1:9876 -t orders  # 各队列的 maxOffset（写入到哪了）

# 建 Topic（生产别依赖自动创建：队列数、权限一次定清）
mqadmin updateTopic -n 127.0.0.1:9876 -b 127.0.0.1:10911 -t orders -r 8 -w 8

# 看堆积（排障第一命令）：Diff 是积压量，按组看
mqadmin consumerProgress -n 127.0.0.1:9876 -g order-consumer-group

# 按 Key 查一条消息（对账、查「这单到底发没发」）
mqadmin queryMsgByKey -n 127.0.0.1:9876 -t orders -k order-1001
mqadmin queryMsgById -n 127.0.0.1:9876 -i <msgId>
```

```java
// 发送三种方式：同步（要确认，重要消息）/ 异步回调（要确认但不阻塞）/ 单向（日志类，丢了无所谓）
// 同步
SendResult r = producer.send(msg);
System.out.println(r.getSendStatus());   // SEND_OK 才算进了 Broker

// 异步：回调里必须处理异常，异常分支别只打日志不管
producer.send(msg, new SendCallback() {
    public void onSuccess(SendResult result) { }
    public void onException(Throwable e) { /* 记失败表，定时补发 */ }
});

// 消费起点要显式选，别用默认蒙对
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET); // 接上次进度（默认）
```

```properties
# broker.conf 可靠性相关的两个基础开关（概念要懂，别乱改）
flushDiskType=ASYNC_FLUSH      # SYNC_FLUSH=每条刷盘才返回（慢但不丢）；ASYNC=靠操作系统刷（快，宕机可能丢最近的）
brokerRole=ASYNC_MASTER        # 主从复制：SYNC_MASTER=等从库确认才返回，与刷盘开关是两个维度
```

**基础参数速览（客户端高频项，先认识再调）**

| 参数 | 在哪 | 含义 |
|---|---|---|
| namesrvAddr | 生产/消费端 | NameServer 地址，多个用分号分隔；配错直接 No route info |
| sendMsgTimeout | 生产端 | 同步发送等回执的上限，默认秒级；消息大或网络差才调大 |
| retryTimesWhenSendFailed | 生产端 | 同步发送失败内部重试次数，重试会换队列发 |
| consumeThreadMin / Max | 消费端 | 消费线程池大小，堆积时先查单条耗时再考虑调大 |
| consumeMessageBatchMaxSize | 消费端 | 一次拉给监听器的最大条数，做批处理才调 |
| pullBatchSize | 消费端 | 每次从 Broker 拉多少条，和批量消费是两层概念 |

Push 消费的真相：`DefaultMQPushConsumer` 名字叫推、底层是客户端自己长轮询拉取（pull），Broker 不主动连你的机器；所以消费者不需要暴露端口，防火墙只放出方向即可。

发送状态（SendStatus）速览：`SEND_OK` 进了 Broker 才算数；`FLUSH_DISK_TIMEOUT`、`FLUSH_SLAVE_TIMEOUT`、`SLAVE_NOT_AVAILABLE` 都是「发出去了但可靠性环节超时/缺席」，重要消息见到这三种要按失败处理或告警，不能当成功。

### 关键机制速览

一条消息的旅程（排障按这条链逐段定位）：

1. Producer 启动时向 NameServer 拉 Topic 路由（哪个 Broker 有哪些队列），之后定时刷新。
2. 发送时按负载均衡挑一个队列，把消息发给对应 Broker。
3. Broker 把消息顺序写进 CommitLog，按 flushDiskType 决定何时刷盘，回执发回 Producer。
4. Consumer 启动时同样拉路由，订阅的队列被分配到组内各成员。
5. Consumer 长轮询拉消息，处理后返回消费状态，Broker 据此推进度。
6. 任一步对不上号：先查路由（NameServer 地址/Topic 是否存在），再查 Broker 状态，最后查消费进度和堆积。

- 存储一句话：消息进 Broker 先顺序写进 CommitLog 一个大文件流，再异步把「队列位置」索引进 ConsumeQueue，消费时按索引回 CommitLog 捞正文——顺序写盘是吞吐的根。
- 可靠性是四个环节的乘积：发送确认（SEND_OK）× 刷盘（同步/异步）× 复制（同步/异步）× 消费确认（返回成功状态才推进度）；任一环是异步/不确认，端到端就不是「不丢」。
- 消费失败自动重试：并发消费返回 RECONSUME_LATER 后，消息进重试队列 `%RETRY% + 消费组`，按递增间隔重投，默认最多重试 16 次（可在消费者端配置），耗尽进死信。推论：消费逻辑先保证幂等，再谈重试。
- 负载均衡是自动的但有代价：消费组内成员增减会触发队列重新分配（rebalance），分配中的消息可能被两个消费者同时处理——又一条「消费必须幂等」的理由。

### 新手易错点

- 消费不做幂等：重试、rebalance、位点重置都会造成重复投递，消费者必须用业务主键（订单号 + 状态机/去重表）挡重复，只靠「一般不会重复」迟早出事。
- 两个不相干业务共用一个消费组：组内是分摊关系，会互相抢消息、进度互相影响；一组一业务，组名带业务前缀。
- 消费里做慢活又不自知：单条消费里调慢接口、跑大 SQL，堆积先涨、最后全进重试和死信；堆积告警响了先看单条消费耗时，别先扩容消费者。
- 发送只判「没抛异常」：send 不抛异常不等于成功，要看 SendResult 的状态；异步和单向方式更要单独设计失败兜底。
- Key 不设或乱设：发送时不带业务 Key，事后按订单号查不到这条消息，对账和排障全靠翻日志，成本翻十倍。
- Topic 靠自动创建：默认 4 队列且各 Broker 不一致，压测和扩容时才发现队列数不对；生产 Topic 走 mqadmin 显式建。
- 重置消费位点当日常操作：危险操作，重置期间消费者还在跑会丢/重叠消费，标准动作是先停消费者再重置（操作前确认影响范围）。

## 实战案例（项目沉淀）

- [翼安反诈平台](项目沉淀/翼安反诈平台.md)：RocketMQ 业务消息实战

## 小技巧与冷知识

【冷知识】ConsumeQueue 里每条索引只有 20 字节：8 字节 CommitLog 物理偏移 + 4 字节消息大小 + 8 字节 Tag 哈希值，单个 ConsumeQueue 文件固定存 30 万条，满了就滚动新文件。索引极小才能常驻内存，这就是原文说的“单机撑 5 万队列不掉性能”的底气——队列一多，拼的不是磁盘，是索引占用的内存。

【冷知识】CommitLog 的文件名就是该文件的起始物理偏移，用 20 位数字补零命名，比如 `00000000001073741824` 表示从 1GB 处开始。排障时看到文件名就能直接算出这段文件覆盖的消息偏移范围，不用逐个打开猜；这也是 CommitLog 只能追加、不能原地改的直接体现。

【小技巧】Tag 过滤在 Broker 端只是粗筛：ConsumeQueue 里存的是 Tag 的哈希值，两个不同 Tag 撞上同一个哈希时，Broker 会把不匹配的消息也推下来，客户端收到后会再过滤一次。所以 Tag 只适合做“大类分流”，别拿它做精确的业务判断；要精确过滤就用消息属性 + SQL92，或者在业务层再判一次。

【小技巧】除了 Tag，RocketMQ 还支持 SQL92 属性过滤订阅，比如按 `age > 18 AND region = '华南'` 这样的表达式选消息。注意 Broker 默认没开这个能力，需要把 `enablePropertyFilter` 设为 true 才生效；同一个 Topic 下不同消费组按属性分流时，比拆多个 Topic 省事，也比全量拉下来再过滤省带宽。

【冷知识】NameServer 不存任何数据：所有路由信息都在内存里，重启后靠 Broker 每 30 秒一次的心跳重新注册回来。所以 NameServer 可以随时重启、随时加节点，不用备份数据；但反过来，生产者和消费者侧一定要配多个 NameServer 地址（分号分隔），只配一个等于给自己留了单点。

【小技巧】单条消息的消费默认 15 分钟超时（`consumeTimeout`），超时会被当作消费失败重投。如果消费逻辑里要调慢接口、等外部回调，千万别在监听器里干等——正确姿势是先落库或转内部任务再返回成功，把长耗时移出消费线程；否则一条慢消息会反复重投，堆积越滚越大。

【小技巧】消息默认只保留 72 小时（`fileReservedTime`），每天凌晨 4 点清理过期文件。想重置位点重放消息，只能在保留期内操作，超过 72 小时消息已经被删了，位点重置回去也捞不回来；磁盘水位告警一定要留提前量，别等到磁盘满了触发 Broker 拒绝写入才处理。

【冷知识】广播消费的位点存在消费者本地文件里（`~/.rocketmq_offsets/offsets.json`），不在 Broker 端。换一台机器部署、或者误删了这个文件，广播消费者就会按配置的起点重新消费；集群消费的位点在 Broker 端则没这个问题。排查“广播消费怎么又从头消费了”时，先看这个本地文件还在不在。

【小技巧】消费失败的重试间隔复用的是那 18 档延迟等级：第一次 1 秒、第二次 5 秒、第三次 10 秒、第四次 30 秒，之后按分钟逐档拉长。一条注定失败的消息走完 16 次重试要拖一个多小时才进死信，所以死信队列的告警要单独做，别等重试走完才发现问题；也可以用 `getReconsumeTimes()` 在消费逻辑里判断重试次数，重试几次后直接记失败表、提前止损。

【小技巧】顺序消息扩容队列数会打乱路由：消息按业务 key 取模队列数决定去向，队列数从 8 扩到 16 之后，同一个 key 很可能被分到另一个队列，历史顺序保证就被打破了。顺序 Topic 扩容前要先评估影响，扩容吞吐优先考虑加 Broker 或拆 Topic，别直接改队列数。

【小技巧】批量发送 `send(Collection<Message>)` 只支持普通消息：要求同一 Topic、总大小不超过上限，延迟消息、顺序消息、事务消息都不能批量。而且批量只是减少网络往返，并不是原子提交，一条失败不影响其他条，别把它当成“要么全发要么全不发”的事务用。

【冷知识】事务消息的半消息对消费者完全不可见：它实际写在一个系统 Topic（`RMQ_SYS_TRANS_HALF_TOPIC`）里，Commit 之前不会出现在业务 Topic 下。所以在控制台按业务 Topic 查不到刚发的事务消息是正常的，先查本地事务执行到哪一步，再看 Broker 回查记录。

【小技巧】同步发送失败的内部重试会换一个队列发，不是原地重发同一队列。这意味着“重试 3 次”期间消息落点会变，顺序消息别指望靠发送重试保序；同时这也是重试能绕开单队列故障的原因，理解了它就不会在排查时对“消息怎么跑到别的队列去了”感到意外。

【小技巧】消费失败返回 `RECONSUME_LATER` 后，消息是进重试队列等下一轮重投，原队列继续往前消费，不会被堵住；顺序消费是例外，`MessageListenerOrderly` 里失败会挂起整个队列。区分这两者很重要：并发消费堆积时可以慢慢消化重试队列，顺序消费堆积时必须先解决卡住的那一条。

【冷知识】按 Key 查消息走的是 Broker 端的 IndexFile，不是扫 CommitLog：Broker 会给每条消息的 Key 建哈希索引，`queryMsgByKey` 查的是这份索引。因为是哈希索引，Key 冲突或哈希冲突时可能一次查到多条，查到了要按业务 Key 再核对一次，别直接拿第一条当结果用。

【小技巧】消费者拉下来的消息会先缓存在客户端内存里，缓存满了就会暂停拉取。所以堆积时别只盯着 Broker 端的 Diff 数字，也要看客户端侧的缓存水位；大消息体多、批量拉取条数大时，客户端缓存最先被打满，表现就是“Broker 有消息、消费者却不拉了”。

## 面试与自测

1. RocketMQ 事务消息的半消息二阶段是怎样走的？Broker 长时间没收到确认会怎样？
2. 4.x 与 5.x 的延迟消息有什么区别？4.x 下想发“17 分钟后”的消息怎么办？
3. 顺序消费里抛异常为什么危险？顺序消息的正确姿势是什么？

### 参考答案

1. 三步：Producer 发 Half 消息（暂不投递）；执行本地事务；发 Commit/Rollback。Broker 长时间没收到确认会回查（默认 15 次），保证最终一致，适合“本地 DB 事务 + 发消息”场景。回查方法只做轻量查询（查本地事务表），别放重逻辑；本地事务表必须落库，不能放内存。
2. 4.x 只有 18 个固定延迟等级（1s~2h）；5.x 基于时间轮实现任意精度定时消息。4.x 下没有“17 分钟”这一档，只能选最接近的固定等级（如 10m 那档）或自己轮询；延迟消息消费失败进死信后延迟语义就没了，消费逻辑要幂等。
3. `MessageListenerOrderly` 中异常会导致该队列阻塞，务必 try-catch 并记录。顺序消息 = 分区有序：全局有序靠单队列，吞吐低；一般用业务 key 分区有序就够了，即 `MessageQueueSelector` 按业务键路由到同一 queue + `MessageListenerOrderly` 单队列单线程消费；有序消费时一个消息卡住会堵住整个队列，消费逻辑必须幂等 + 快速失败。
