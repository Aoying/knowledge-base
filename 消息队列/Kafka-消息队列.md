---
知识点: Kafka 消息队列
层级: 进阶
适用版本: Kafka 4.x
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】key 为 null 的消息不是随机进分区，默认走粘性分区器：Kafka 2.4 起默认分区器对 null key 用 StickyPartitioner，一段时间内粘在同一个分区攒批，攒满一批再换下一个分区。这样既打散了数据，又比逐条轮询少了大量小批次；如果你在压测里看到 null key 消息成批落在同一分区，不用怀疑是分区器坏了，这是刻意为之的优化。

【小技巧】排障时让 console-consumer 把 key 和时间戳一起打出来：`kafka-console-consumer.sh --property print.key=true --property print.timestamp=true --property key.separator="|"`，一眼就能看出这条消息进了哪个分区键、时间戳是生产者写入时间（CreateTime）还是 broker 落盘时间（LogAppendTime）。只看 value 的排障等于蒙着一半眼睛，尤其是怀疑“同一用户消息进了不同分区”时，key 必须可见。

【小技巧】compact 策略的 topic 里，value 为 null 的消息是墓碑：log compaction 保留每个 key 的最新值，但 value 为 null 的记录表示“删除这个 key”，清理线程会在 `delete.retention.ms`（默认 24 小时）后把墓碑和旧值一起清掉。往 compact topic 里写 null value 不是写了条空消息，是在删数据；想保留“空值”这个状态本身，别用 compact topic。

【小技巧】segment 文件可以直接用 `kafka-dump-log.sh` 看，不用猜：`kafka-dump-log.sh --files /data/kafka/orders-0/00000000000000000000.log --print-data-log` 能看到段里每条记录的 offset、时间戳、key 和 value。怀疑“消息到底写没写进磁盘”时，直接看段文件比在客户端层面反复收发更可信；段文件配套的 `.index` 和 `.timeindex` 分别是 offset 索引和时间戳索引，查旧消息先走索引、不用全段扫描。

【小技巧】改 topic 的保留期不用重建，用 `kafka-configs.sh` 动态改：`kafka-configs.sh --bootstrap-server localhost:9092 --alter --entity-type topics --entity-name orders --add-config retention.ms=259200000`，改完立即生效，新段按新保留期清理。`retention.ms` 和 `retention.bytes` 是“先到先清”的关系，时间到了整段删、大小超了也整段删，两个都设时以先触发的为准。

【小技巧】副本掉出 ISR 先看 `replica.lag.time.max.ms`，而不是先加机器：副本超过这个时间（默认 30 秒）没跟上 leader 的写入就会被踢出 ISR，`--describe` 里 Isr 变少往往是这个原因。网络抖动、GC 停顿都可能让副本短暂落后，被踢出后它追上进度会自动回 ISR；反复被踢再查磁盘 IO 和网络，而不是一上来就扩副本数。

【小技巧】新客户端连老 broker 一般能跑，老客户端连新 broker 未必行：Kafka 的兼容策略是“新客户端兼容老 broker”，协议向前兼容；但 4.x 的 KRaft 集群要求客户端支持较新的 wire protocol，太老的客户端直接连不上。升级顺序永远是先升客户端、再升 broker，原文 4.x 变化里说的“太老的客户端连不上 4.x”就是这个规则的落地。

【小技巧】KRaft 集群的元数据可以用 `kafka-metadata-shell.sh` 直接看：`kafka-metadata-shell.sh --snapshot /data/kafka/__cluster_metadata-0/00000000000000000000.log` 能读出 controller quorum 里存的 topic、分区和副本分配，不用等集群起来再 `--describe`。集群起不来、怀疑元数据损坏时，这是第一手证据；但它读的是快照文件，看到的只是某个时间点的元数据状态。

【小技巧】`kafka-storage.sh format` 的 cluster id 一台机器只能格式化一次：格式化后 cluster id 写进数据目录，换一个新 id 再 format 会报错，必须先清空数据目录。本地单机换个干净环境重建时，删数据目录和换 cluster id 要一起做，只换 id 不清目录等于白换。

【小技巧】扩副本、搬分区用 `kafka-reassign-partitions.sh`，跟进度用 `--verify`：生成重分配 JSON 后执行，再用 `--verify` 反复看每个分区的搬迁状态，全部完成前别重启 broker。搬分区期间 leader 选举会短暂抖动，业务高峰别做；`kafka-leader-election.sh --election-type preferred` 可以在搬完后手动把 leader 触发回首选副本，恢复读写均衡。

【小技巧】`auto.commit.interval.ms` 默认 5 秒，位移提交是“定期”不是“每条”：自动提交下消费者每 5 秒批量提交一次已 poll 的 offset，中间崩了会从上次提交点重读，最多重复 5 秒的量。把这个值调小能缩小重复窗口，但提交本身也是写 `__consumer_offsets` 的开销；`__consumer_offsets` 默认 50 个分区，消费组一多它就是高频写入的内部 topic，原文说的“别手贱去删”之外，还要知道它本身也会成为写入热点。

【冷知识】Kafka 的名字来自作家弗兰兹·卡夫卡：LinkedIn 的 Jay Kreps 觉得 Kafka 的写作风格（荒诞、系统性的官僚）很像分布式系统，就借了这个名字。项目 2011 年在 LinkedIn 开源、2012 年成为 Apache 顶级项目，最初只是 LinkedIn 内部的日志管道，后来才长成今天的流平台。

【冷知识】KRaft 的全名来自 KIP-500 的 “Kafka Raft Metadata mode”：它不是把 ZooKeeper 的功能原样搬进 Kafka，而是把元数据事件本身也写成一条日志（`__cluster_metadata`），controller quorum 用 Raft 协议对这条日志达成共识。也就是说 Kafka 用“自己的日志”管“自己的元数据”，元数据和业务消息第一次用上了同一套存储思想。

【冷知识】Kafka 快的秘密有一半在操作系统不在自己手里：顺序写 + 页缓存的意思是，写进来的数据大概率还留在 OS 的 page cache 里，读的时候根本不用碰磁盘。所谓“磁盘顺序写比随机读内存快”只在页缓存命中时成立，内存给少了、页缓存装不下热数据，Kafka 也会慢下来——原文“内存比 CPU 更重要”的推论，根子就在这里。

【冷知识】零拷贝 `sendfile` 在 Linux 上才真正零拷贝：Kafka 用 `FileChannel.transferTo` 把段文件直接送进 socket，数据不经过 JVM 堆；在 macOS、Windows 等系统上它会退化成普通的读 + 写拷贝，性能优势打折。所以同一套压测数字换了操作系统就不能直接套用，基准测试的机器环境本身就是变量之一。

【冷知识】分区数“只增不减”还有一层原因在 segment 文件上：分区数翻倍后，broker 上每个分区的 segment 文件、索引文件和文件句柄也跟着翻倍，`__consumer_offsets` 里每个消费组要跟踪的 offset 也翻倍。原文说 controller 负担重，落到磁盘上就是“每个分区都是一组真实文件”，分区不是逻辑数字，是实打实的 IO 与句柄成本。

## 面试与自测

1. 为什么说 Kafka 的消费是“拿着书签往后读”，和传统队列“取出即删”有什么根本不同？
2. 分区数定多少合适？为什么说分区数是“只增不减”的决策？
3. `--describe` 输出里 Replicas 比 Isr 多说明什么？应该先查什么？

### 参考答案

1. Kafka 是分布式提交日志：消息按顺序追加写进分区尾部、永不修改，消费就是拿着书签（offset）从某个位置往后读，消费进度存在内部 topic `__consumer_offsets` 里；读过不删，到保留期才整段清掉。和传统队列“取出即删”是两种世界观。
2. 分区数 ≥ 消费线程数，不然加机器也白加；但别一上来就 100 个分区，controller 负担重。分区数只能调大，调大后 key 的归属会变，保序要求高的业务扩分区前先评估 key 重映射影响，必要时停机迁移。
3. 说明有副本掉队：`Leader` 是当前干活的副本，`Replicas` 是全部副本，`Isr` 是跟上进度的副本；Replicas 比 Isr 多时先查那台掉队的 broker。
