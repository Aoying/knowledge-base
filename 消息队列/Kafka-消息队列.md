---
知识点: Kafka 消息队列
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-22
关联:
  - 数据库知识点地图
  - RocketMQ
标签:
  - 消息队列
  - 实践
---

# Kafka 消息队列

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

## 时效核验小结（2026-10-20）

- 笔记里"4.x 已彻底去掉 ZooKeeper，全面 KRaft"——成立。4.0.0（2025-03-18）移除 ZK，4.x 只有 KRaft 一种模式；老集群升级路径是 3.x(ZK) → 3.9(KRaft) → 4.x。
- "Share Groups（4.2 起生产可用）"——成立。4.1 为预览，4.2（2026-02）生产可用；另有 KIP-848 新 rebalance 协议 GA，重平衡更快更增量。
- 本地单机 compose 里的 `apache/kafka:4.3.1` 镜像——存在，是官方镜像（2026-09 时的最新稳定版）；4.4.0 RC 正在投票，还没 GA，单机验证没必要追。
- broker 需要 Java 17+（client 需 Java 11+）；4.0 起旧 wire protocol 和 v0/v1 消息格式被移除，太老的客户端连不上 4.x broker。

## Spring Kafka 死信队列 DLT 实战

笔记的消费片段里写了"别 ack，下次重投；记死信"，但没给死信落地做法。生产标准做法：

```java
@Configuration
public class KafkaErrorConfig {
    @Bean
    public DefaultErrorHandler kafkaErrorHandler(KafkaTemplate<String, String> kafkaTemplate) {
        // 重试耗尽后发往 <原topic>.DLT（默认后缀），带原始 topic/partition/offset/异常堆栈等 header
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
        // 指数退避：1s, 2s, 4s, 8s（封顶10s），共 5 次尝试
        ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(4);
        backOff.setInitialInterval(1_000L);
        backOff.setMultiplier(2.0);
        backOff.setMaxInterval(10_000L);
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
        handler.addNotRetryableExceptions(ValidationException.class); // 参数校验错重试没意义，直接进 DLT
        return handler;
    }
}

@Component
public class OrderConsumer {
    @KafkaListener(topics = "orders", groupId = "g1")
    public void on(String msg) { process(msg); }   // 抛异常走上面的 handler

    @DltHandler
    public void dlt(String msg,
            @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String topic,
            @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String errMsg) {
        log.error("进死信 orders.DLT, 原topic={}, 原因={}", topic, errMsg);
        alarmService.notify(...);   // 死信必须告警+可重放，别只记日志
    }
}
```

- DLT topic 默认分区数**至少**要和原 topic 一样（默认按原分区号投递），否则改 destinationResolver 做取模映射。
- 重试总耗时别超过 `max.poll.interval.ms`（默认 5 分钟），否则重试中途触发 rebalance。
- DLT 不是垃圾桶：保留期设得比原 topic 长，配"死信重放"脚本（读 DLT → 重发原 topic），故障恢复时用得上。
- 批量消费（`List<ConsumerRecord>`）同样配 `DefaultErrorHandler`，失败的那条进 DLT、offset 照常提交，不卡住整批。

## Topic 创建与服务端关键参数实战

```bash
# 建 topic：分区数>=消费线程数，副本数>=3 的集群用 3
kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic orders --partitions 12 --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000        # 7天，默认也是7天，显式写出来防"默认"背刺

# 看 topic 配置 / 加分区（只能调大）
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic orders --partitions 24
```

```properties
# broker server.properties：生产必改
unclean.leader.election.enable=false  # 宁可短暂不可用，不丢数据（笔记疑难杂症1已讲）
min.insync.replicas=2                 # 和 acks=all 配套
auto.create.topics.enable=false       # 生产关掉，topic 走评审建，防打字错误建出一堆 topic
log.retention.ms=604800000
```

- `min.insync.replicas=2` + `acks=all` + `enable.idempotence=true` 是"不丢"的铁三角，缺一个都不完整。
- 分区数是"只增不减"的决策：12 分区起步够中小业务；key 保序的业务扩分区前先评估 key 重映射影响（笔记"分区 key 设计"一节已讲）。
- `__consumer_offsets` 别动是对的；另外 `auto.create.topics.enable=true` 是本地省事、生产埋雷的典型，关掉。

> 来源（2026-10-20 核验）：[kafka-4.x-docker](https://www.webcodingcenter.com/apache-kafka/installing-apache-kafka-4-x-in-docker-with-kraft--kafka-4-x-docker-images-and-distributions.html)；[kafka-digest-2026-09](https://developers.redhat.com/blog/2026/10/05/kafka-monthly-digest-september-2026)；[spring-kafka-error-handling](https://docs.spring.io/spring-kafka/reference/3.3/kafka/annotation-error-handling.html)

## 待补充





## 疑难杂症

### 1. ISR 收缩与 unclean leader election 丢数据
- **现象**：broker 挂了一个，恢复后发现少了一段消息。
- **原因**：`unclean.leader.election.enable=true` 时，非 ISR 副本也能当选 leader，它缺的数据就丢了。
- **解决**：生产环境设 `unclean.leader.election.enable=false`，宁可短暂不可用也不丢数据；`min.insync.replicas` 至少 2。

### 2. 消费者 lag 持续增长的排查链路
- **现象**：监控看 lag 只增不减，消费组像没在干活。
- **原因**：按顺序查：消费者是不是挂了/rebalance 不停 → 单条消费耗时（下游慢）→ 分区数 < 消费者线程数（扩线程没用）→ broker 磁盘/网络瓶颈。
- **解决**：先看消费耗时，90% 的 lag 都是下游慢；确认分区数 >= 消费线程数；临时扩分区+消费者顶过去再查根因。

### 3. exactly-once 的代价
- **现象**：开了幂等+事务，吞吐掉了 30%+，延迟也高了。
- **原因**：exactly-once = 幂等生产者 + 事务 + 消费者幂等处理，三重成本。
- **解决**：不是所有业务都要 exactly-once；能接受"至少一次+消费端幂等"就别开事务；真要开，压测出基准再上线。

## 本地单机 KRaft（2026-10-18 核验：Kafka 4.0 已移除 ZooKeeper，KRaft 是唯一模式）

```yaml
# docker-compose.yml
services:
  kafka:
    image: apache/kafka:4.3.1
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
```
```bash
KAFKA_CLUSTER_ID=$(docker exec kafka /opt/kafka/bin/kafka-storage.sh random-uuid)
docker exec kafka /opt/kafka/bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /opt/kafka/config/server.properties
docker compose up -d
```

- 单机把 broker 和 controller 合在一个进程（`broker,controller`），开发够用，生产要拆开。
- 来源：https://www.webcodingcenter.com/apache-kafka/event-streaming-concepts-and-the-publish-subscribe-log--kafka-4-x-and-the-kraft-only-release-line.html

## 分区 key 设计

```java
// 按业务主键分区：同一用户的消息永远进同一分区 → 分区内有序
producer.send(new ProducerRecord<>("orders", userId, orderJson));
```

- **key 选谁**：需要保序的维度就是 key（用户 ID、订单 ID）；不需要保序就别设 key，让它轮询打散。
- **分区数**：`分区数 >= 消费线程数`，不然加机器也白加；但别一上来 100 个分区，controller 负担重。
- 改分区数只能调大不能调小（调大后 key 的归属会变，保序要求高的业务要停机迁移）。
- 选型对比见待补充：RocketMQ vs Kafka（`消息队列/RocketMQ.md` 的顺序消息可对照看）。

## Spring Kafka 实战片段

```java
// 生产
kafkaTemplate.send("orders", userId, orderJson);

// 消费：批量 + 手动提交（推荐生产配置）
@KafkaListener(topics = "orders", groupId = "g1")
public void onBatch(List<ConsumerRecord<String,String>> rs, Acknowledgment ack) {
    try { process(rs); ack.acknowledge(); }
    catch (Exception e) { /* 别 ack，下次重投；记死信 */ }
}
```

```yaml
spring.kafka.consumer:
  enable-auto-commit: false
  max-poll-records: 50
  properties.max.poll.interval.ms: 300000
```

- 关掉自动提交，业务成功才 `ack`，这是"至少一次"的底线。
- `max.poll.interval.ms` < 单批处理耗时会触发 rebalance，大批量调大它或调小 `max-poll-records`。

## RocketMQ vs Kafka 选型对比

|  | RocketMQ | Kafka |
|---|---|---|
| 出身 | 阿里，双十一 | LinkedIn，日志流 |
| 顺序消息 | 原生支持（顺序队列） | key 分区保序 |
| 事务消息 | 原生（half message+回查） | 事务 API（偏重 exactly-once） |
| 延迟消息 | 原生 18 级延迟 | 要自己实现/4.2+ share group |
| 运维 | NameServer 轻量 | KRaft（4.0+ 去 ZK） |
| 生态 | 国内金融/电商多 | 大数据生态（Flink 等） |

- 神农 e 贷选 RocketMQ 是对的：金融要事务消息+延迟消息，开箱即用。
- 日志/埋点/流计算选 Kafka：和 Flink/Spark 一家。

## 监控关键指标

| 指标 | 含义 | 告警线 |
|---|---|---|
| UnderReplicatedPartitions | 副本没跟上的分区数 | >0 就查 |
| Consumer Lag | 消费落后 | 持续增长告警 |
| RequestHandlerAvgIdlePercent | broker 空闲率 | <0.3 说明 broker 快顶不住了 |
| LogEndOffset 增速 | 写入速率 | 突增查生产者 |

- JMX Exporter + Prometheus + Grafana 一套，Kafka 官方推荐链路。
- Lag 告警配"持续 5 分钟增长"而不是"绝对值"，避免正常波动误报。

## 基础知识补强

### 核心概念与术语

- 心智模型：Kafka 是「分布式提交日志」——消息按顺序追加写进分区尾部，永不修改，消费就是拿着书签（offset）从某个位置往后读，读过不删，到保留期才整段清掉。和传统队列「取出即删」是两种世界观。
- 记录（record）的结构：key（决定进哪个分区，可为空）+ value（正文）+ timestamp + headers（附带元数据，如 traceId）；业务数据放 value，链路信息放 header，别把正文塞进 key。
- 副本与 ISR：每个分区一个 leader 处理读写，若干 follower 被动复制；与 leader 保持同步的副本集合叫 ISR，leader 挂了只能从 ISR 里选新 leader。副本数（replication factor）= 数据有几份，ISR 是「跟上进度的那几份」。
- 投递语义三档（记定义，别只记名字）：at-most-once（至多一次，可能丢）、at-least-once（至少一次，可能重复，Kafka 默认实际效果）、exactly-once（恰好一次，要幂等+事务+消费端配合才成立）。绝大多数业务正确答案是 at-least-once + 消费端幂等。
- 日志段（segment）：分区在磁盘上按段滚动存文件，写满或到期换新段；清理按段整段删（delete 策略）或按 key 压缩保留最新值（compact 策略，配置/状态类 topic 用）。
- 消费者坐标系：一个消费组订阅一个 topic 时，分区是分配单位；组名（group.id）就是身份，改了组名等于换了身份，位移记录对不上，默认从最新或最早重新开始。

### 基础语法与用法

```bash
# 命令行四件套（本地调试、排障都靠它们；--bootstrap-server 填任一 broker 即可，它会自己发现全集群）

# 1) 造数据：控制台生产者（每回车一条；带 key 用第二种，key 和 value 用冒号分隔要加 parse.key）
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders \
  --property "parse.key=true" --property "key.separator=:"

# 2) 看数据：--from-beginning 从最早读；不加默认只读新消息（新手最常踩）
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders --from-beginning
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders \
  --property print.key=true --max-messages 10

# 3) 查消费进度：lag 看这一列（组在消费哪个 topic、每个分区落后多少）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group g1

# 4) 重置位移（危险操作：先停消费者再执行，要加 --dry-run 先预览）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group g1 \
  --reset-offsets --to-earliest --topic orders --dry-run
```

```java
// 原生客户端最小骨架（理解 Spring Kafka 封装之前先看懂这层）
Properties p = new Properties();
p.put("bootstrap.servers", "localhost:9092");          // 集群入口列表，写 2~3 个防单点
p.put("group.id", "g1");                               // 消费者必填：组身份
p.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
p.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
p.put("enable.auto.commit", "false");                  // 先关自动提交，处理成功再手动 commitSync

KafkaConsumer<String, String> c = new KafkaConsumer<>(p);
c.subscribe(List.of("orders"));                        // 订阅即加入组，分区自动分配
while (true) {
    ConsumerRecords<String, String> rs = c.poll(Duration.ofMillis(500)); // 长轮询拉一批
    for (ConsumerRecord<String, String> r : rs) {
        process(r.value());                            // r.partition()/r.offset() 是定位三件套
    }
    c.commitSync();                                    // 整批成功后才提交位移
}
```

```java
// 原生生产者最小骨架（和消费者对称，先看懂再用 Spring 封装）
Properties pp = new Properties();
pp.put("bootstrap.servers", "localhost:9092");
pp.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
pp.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
pp.put("acks", "all");
KafkaProducer<String, String> producer = new KafkaProducer<>(pp);
// 异步发 + 回调确认：get() 阻塞等结果是同步发，新手排障可用，业务主链路别这么写
producer.send(new ProducerRecord<>("orders", userId, orderJson), (meta, ex) -> {
    if (ex != null) { /* 发送失败兜底：记失败表重试 */ }
    else { /* meta.topic()/partition()/offset() 就是这条消息的最终坐标 */ }
});
producer.close();   // 退出前必须 close/flush，没发完就关进程会丢消息
```

清理策略二选一（建 topic 时按数据性质定）：

| 策略 | 配置 | 行为 | 适合 |
|---|---|---|---|
| delete | cleanup.policy=delete | 到保留期/保留量整段删 | 订单、日志等流水型消息（默认） |
| compact | cleanup.policy=compact | 同 key 只留最新一条 | 配置下发、实体最新状态（如商品价格） |

分区副本读图法：`kafka-topics.sh --describe` 输出里 `Leader` 是当前干活的副本，`Replicas` 是全部副本，`Isr` 是跟上进度的副本；Replicas 比 Isr 多，说明有副本掉队，先查那台 broker。

### 关键机制速览

- 为什么快，三件套：顺序追加写磁盘（避开随机 IO）+ 页缓存（数据先写 OS cache，读也走 cache，不吃 JVM 堆）+ 零拷贝 sendfile（文件直接进网卡，不在用户态来回搬）。推论：给 Kafka 的机器内存比 CPU 更重要，堆别调太大，留给页缓存。
- 位移存在哪：消费组的位移存在内部 topic `__consumer_offsets`，不是客户端本地文件；所以换机器消费能接上进度，改组名则接不上。
- 拉模式（pull）而非推模式：消费者按自己的节奏 poll，天然削峰；代价是要自己处理「没数据时别空转」（poll 本身是长轮询，不用再 sleep 忙等）。
- 分区内有序、分区外无序：同一分区的 offset 就是全局顺序的定义；跨分区没有先后可言，下单类消息想有序必须同 key 进同分区（或业务侧带序号自理）。

### 新手易错点

- 控制台消费「什么都看不到」：默认从最新位移开始，只收启动之后的新消息；看存量加 `--from-beginning`，这不是丢消息。
- bootstrap.servers 只写一个地址还以为高可用：它只是初始入口名单，写的那台恰好宕机时新客户端连不上集群，常态写 2~3 台。
- key 传 null 还指望有序：null key 走轮询/粘性分配打散到各分区，顺序无从谈起；要保序必须给业务 key。
- 靠改大 retention 以为能「找回」被删数据：保留到期整段删除，删了就是删了，延长保留期只能从设置之后生效，存量找不回。
- 随手改 group.id 上线：新组没有历史位移，按 auto.offset.reset（默认 latest）从尾巴开始，等于跳过全部存量；要重消费必须明确指定并评估下游幂等。
- poll 循环里做长耗时同步调用又不调参数：单批处理超过 max.poll.interval.ms 会被判定掉线踢出组引发 rebalance，先算单批最长耗时再配置。
- 在分区数上贪多求快：分区数是并行上限也是元数据负担，几十个分区和几千个分区是两回事，按消费并行需求定，别给每个小 topic 都 100 分区。
