---
知识点: Kafka 消息队列
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-23
关联:
  - RocketMQ
  - Kafka-消费者组与Rebalance
  - Kafka-生产者可靠性
  - 消息可靠性
  - Kafka与RocketMQ对比
标签:
  - 消息队列
  - 实践
---

# Kafka 消息队列

> 本篇是 Kafka 的汇总与索引：全景、核心概念、选型与服务端要点留在本篇；生产端、消费端的细节已拆为独立篇（见下方索引），正文不再重复。时效性以 Apache Kafka 官方为准（2026 年主线 4.x，已彻底去掉 ZooKeeper，全面 KRaft）。

## 全景导读

- **心智模型**：Kafka 是「分布式提交日志」——消息按顺序追加写进分区尾部、永不修改，消费就是拿着书签（offset）从某个位置往后读，读过不删，到保留期才整段清掉。和传统队列「取出即删」是两种世界观。
- **为什么快**，三件套：顺序追加写磁盘（避开随机 IO）+ 页缓存（读写走 OS cache，不吃 JVM 堆）+ 零拷贝 sendfile（文件直接进网卡）。推论：给 Kafka 的机器内存比 CPU 更重要，堆别调太大，留给页缓存。
- **投递语义三档**（记定义，别只记名字）：at-most-once（可能丢）、at-least-once（可能重复，Kafka 默认实际效果）、exactly-once（要幂等 + 事务 + 消费端配合才成立）。绝大多数业务的正确答案是 at-least-once + 消费端幂等，机制对比与选型清单见 [消息可靠性](消息队列/消息可靠性.md)。
- **拉模式（pull）**：消费者按自己的节奏 poll，天然削峰；poll 本身是长轮询，没数据时不用再 sleep 忙等。

## 核心概念一句话

- **Topic**：消息的逻辑分类；**Partition**：topic 的分片，并行和顺序性的基本单位。
- **Offset**：partition 内的消息序号，消费进度就是 offset，存在内部 topic `__consumer_offsets`。
- **Consumer Group**：一组消费者共同消费一个 topic，一个 partition 同一时刻只被组内一个消费者消费。
- **Broker**：Kafka 服务节点；**KRaft**：4.x 的元数据共识机制（替代 ZooKeeper）。
- **记录（record）结构**：key（决定进哪个分区，可为空）+ value（正文）+ timestamp + headers（附带元数据，如 traceId）；业务数据放 value，链路信息放 header，别把正文塞进 key。
- **副本与 ISR**：每个分区一个 leader 处理读写，若干 follower 被动复制；与 leader 保持同步的副本集合叫 ISR，leader 挂了只能从 ISR 里选新 leader。
- **日志段（segment）与清理策略**：分区在磁盘上按段滚动存文件，清理按段整段删（delete 策略，流水型消息默认）或按 key 压缩只保留最新值（compact 策略，配置/状态类 topic 用）。

## 子篇索引

- [Kafka 生产者可靠性](消息队列/Kafka-生产者可靠性.md)：不丢、不重、不乱序分别靠什么——`acks` 与 `min.insync.replicas` 配套、重试的重复与乱序副作用、幂等生产者的原理与边界、事务消息的概念与成本、发送回调与异常分类。
- [Kafka 消费者组与 Rebalance](消息队列/Kafka-消费者组与Rebalance.md)：分区分配策略（Range / RoundRobin / Sticky / CooperativeSticky）、rebalance 的触发条件与风暴成因、心跳与 poll 间隔参数关系、位点提交的正确姿势、消费失败进死信队列（DLT）的标准做法。

## 相关专题

- [消息可靠性](消息队列/消息可靠性.md)：三档语义边界、Kafka/RocketMQ 机制对比、消费端幂等三种模式、选型决策清单。
- [Kafka 与 RocketMQ 对比](消息队列/Kafka与RocketMQ对比.md)：与 [RocketMQ](消息队列/RocketMQ.md) 的系统性对照。

## 选型要点（vs RocketMQ）

|  | RocketMQ | Kafka |
|---|---|---|
| 出身 | 阿里，双十一 | LinkedIn，日志流 |
| 顺序消息 | 原生支持（顺序队列） | key 分区保序 |
| 事务消息 | 原生（half message+回查） | 事务 API（偏重 exactly-once） |
| 延迟消息 | 原生 18 级延迟 | 要自己实现/4.2+ share group |
| 运维 | NameServer 轻量 | KRaft（4.0+ 去 ZK） |
| 生态 | 国内金融/电商多 | 大数据生态（Flink 等） |

- 神农 e 贷选 RocketMQ 是对的：金融要事务消息 + 延迟消息，开箱即用。
- 日志/埋点/流计算选 Kafka：和 Flink/Spark 一家。

## 分区 key 设计

```java
// 按业务主键分区：同一用户的消息永远进同一分区 → 分区内有序
producer.send(new ProducerRecord<>("orders", userId, orderJson));
```

- **key 选谁**：需要保序的维度就是 key（用户 ID、订单 ID）；不需要保序就别设 key（null key 走轮询/粘性分配打散），还指望有序是最常见的误会。
- **分区数**：`分区数 >= 消费线程数`，不然加机器也白加；但别一上来 100 个分区，controller 负担重。分区数是「只增不减」的决策，调大后 key 的归属会变，保序要求高的业务扩分区前先评估 key 重映射影响，必要时停机迁移。

## Topic 创建与服务端关键参数

```bash
# 建 topic：分区数 >= 消费线程数，副本数 >= 3 的集群用 3
kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic orders --partitions 12 --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000        # 7 天，显式写出来防"默认"背刺

kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic orders --partitions 24  # 只能调大
```

```properties
# broker server.properties：生产必改
unclean.leader.election.enable=false  # 宁可短暂不可用，不丢数据
min.insync.replicas=2                 # 和生产者 acks=all 配套
auto.create.topics.enable=false       # 生产关掉，topic 走评审建，防打字错误建出一堆 topic
log.retention.ms=604800000
```

- 读图法：`--describe` 输出里 `Leader` 是当前干活的副本，`Replicas` 是全部副本，`Isr` 是跟上进度的副本；Replicas 比 Isr 多，说明有副本掉队，先查那台 broker。
- `__consumer_offsets` 膨胀是正常现象，别手贱去删。

## 命令行四件套（排障基本功）

```bash
# 造数据（带 key 要加 parse.key 并用分隔符）
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
# 看数据：不加 --from-beginning 只读启动后的新消息，新手最常在这里以为"丢消息"
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders --from-beginning
# 查消费进度（lag 看这一列）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group g1
# 重置位移（危险操作：先停消费者，先 --dry-run 预览）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group g1 \
  --reset-offsets --to-earliest --topic orders --dry-run
```

- `bootstrap.servers` 只是初始入口名单，写 2~3 个 broker 地址防单点；只写一个、那台恰好宕机时新客户端连不上集群。

## 4.x 值得知道的变化

- **KRaft**：4.0（2025-03）起移除 ZooKeeper，只有 KRaft 一种模式；老集群升级路径是 3.x(ZK) → 3.9(KRaft) → 4.x。broker 需要 Java 17+，太老的客户端（旧 wire protocol）连不上 4.x。
- **Share Groups（4.2 起生产可用）**：队列语义——多消费者可协作消费同一 partition，带单条 ack；适合「既用 Kafka 又要队列语义」的场景，不必为此单独搭 RabbitMQ。4.1 为预览。
- **新 rebalance 协议（KIP-848）**：分配移到 broker 侧、增量协调，rebalance 更快，细节见 [Kafka 消费者组与 Rebalance](消息队列/Kafka-消费者组与Rebalance.md)。

## 监控关键指标

| 指标 | 含义 | 告警线 |
|---|---|---|
| UnderReplicatedPartitions | 副本没跟上的分区数 | >0 就查 |
| Consumer Lag | 消费落后 | 持续增长告警 |
| RequestHandlerAvgIdlePercent | broker 空闲率 | <0.3 说明 broker 快顶不住了 |
| LogEndOffset 增速 | 写入速率 | 突增查生产者 |

- JMX Exporter + Prometheus + Grafana 是官方推荐链路；Lag 告警配「持续 5 分钟增长」而不是绝对值，避免正常波动误报。lag 本身的排查链路在 [Kafka 消费者组与 Rebalance](消息队列/Kafka-消费者组与Rebalance.md) 的疑难杂症里。

## 本地单机 KRaft

```yaml
# docker-compose.yml（开发够用；生产 broker 与 controller 要拆开部署）
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

## 实战案例（项目沉淀）

- [翼安反诈平台](项目沉淀/翼安反诈平台.md)：Kafka 在反诈数据管道中的使用
