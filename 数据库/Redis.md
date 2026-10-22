---
知识点: Redis
层级: 进阶
适用版本: Redis 7
更新日期: 2026-10-22
关联:
  - 数据库知识点地图
  - MySQL-数据库
标签:
  - 数据库
  - 缓存
  - 实践
---

# Redis

> 每晚 20:30 自动补充完善。贴合 Redis 7/8（项目在用 Redis 7/8）。

## 数据结构与典型场景

| 结构 | 场景 |
|---|---|
| String | 缓存、计数器、分布式锁 |
| Hash | 对象缓存（用户信息） |
| List | 队列、时间线 |
| Set | 去重、共同好友 |
| ZSet | 排行榜、延迟队列（score=时间戳） |
| Stream | 消息队列（替代 List 的可靠队列） |
| Bitmap / HyperLogLog | 签到、UV 统计 |

## 常用命令速查

```bash
# String
SET key val EX 3600 NX        # 带过期+仅不存在时设置（分布式锁核心）
GETSET key val                # 原子 get+set
INCR / DECR                   # 计数器
MSET / MGET                   # 批量

# Hash
HSET user:1 name 张三 age 18
HGETALL user:1

# ZSet
ZADD rank 100 "u1" 95 "u2"
ZREVRANGE rank 0 9 WITHSCORES  # 前10名

# 过期与删除
TTL key
DEL key                        # 大 key 用 UNLINK（异步删，不阻塞）
SCAN 0 MATCH "user:*" COUNT 1000  # 生产禁用 KEYS *
```

## 持久化

- **RDB**：定时快照，恢复快，丢最近数据。`save 900 1`（900秒内1次写则快照）。
- **AOF**：记录写命令，`appendfsync everysec`（每秒刷盘，折中之选）。
- 生产一般 RDB+AOF 都开，AOF 优先恢复。

## 缓存三大问题

### 缓存穿透（查不存在的 key，打爆 DB）
- 布隆过滤器前置拦截。
- 空值也缓存：`SET null_key "" EX 60`（短过期）。

### 缓存击穿（热点 key 过期瞬间，大量请求打 DB）
- 热点 key 永不过期（后台异步更新）。
- 互斥锁：只放一个请求去 DB，其他等。

### 缓存雪崩（大量 key 同时过期 / Redis 挂了）
- 过期时间加随机值，打散。
- Redis 集群+哨兵/Cluster 高可用；本地加一级进程内缓存（Caffeine）兜底。

## 分布式锁（重点）

```java
// 1. 加锁：SET key uuid NX PX 30000（值用 uuid 防误删）
// 2. 业务
// 3. 解锁：Lua 脚本比对 uuid 一致才 DEL（保证原子）
String lua = "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
```
- 锁超时 < 业务耗时 → 锁提前释放：用 Redisson 看门狗自动续期，或业务做幂等。
- 要求不高直接 Spring Integration / Redisson；自己手写只记住 **NX+PX+uuid+Lua** 四件套。

## 内存淘汰策略

`maxmemory-policy`：`noeviction`（默认，写满报错）/ `allkeys-lru` / `volatile-lru` / `allkeys-lfu`。
缓存场景一般选 `allkeys-lru`，并设 `maxmemory`（比如 4GB），别让 Redis 吃光机器内存。

## 常见坑

1. **大 key**：单个 key 几十 MB，`DEL` 会阻塞。用 `UNLINK`，或拆分成 hash。
2. **热 key**：单 key QPS 过高打爆单节点 → 本地缓存 + key 加后缀打散。
3. **pipeline**：批量操作一次 RTT，用 pipeline 别循环单条。
4. **连接池**：Lettuce 默认够用，别每个请求 new 连接。
5. **序列化**：Java 项目用 `GenericJackson2JsonRedisSerializer` 或 String，别用 JDK 原生序列化（臃肿且跨语言不可读）。

## 待补充




## 疑难杂症

### 1. 生产环境用了 KEYS * 直接打挂
- **现象**：想查个 key 模式，`KEYS *` 一执行，Redis 整个卡死。
- **原因**：`KEYS` 是 O(N) 全量扫描，单线程模型下直接阻塞所有请求。
- **解决**：一律用 `SCAN` 游标分批扫；生产环境可以 rename 掉 `KEYS` 命令防手滑。

### 2. RDB/AOF 选型与数据恢复
- **现象**：Redis 重启后数据丢了，或恢复巨慢。
- **原因**：纯 RDB 两次快照之间的数据会丢；纯 AOF 文件太大恢复慢；`appendfsync` 策略影响写入性能。
- **解决**：生产用 RDB+AOF 混合持久化；`appendfsync everysec` 是常用折中；定期把 RDB 备份到异机。

### 3. Cluster 模式 key 报错 CROSSSLOT
- **现象**：`MGET`/`MSET` 多个 key 报 `CROSSSLOT Keys in request don't hash to the same slot`。
- **原因**：Cluster 按 key 的哈希槽分片，多 key 操作要求 key 落在同一个槽。
- **解决**：用 hash tag 把相关 key 绑一起：`{user:1}:name`、`{user:1}:age`，花括号内的部分决定槽位。

## Stream 消费者组实战

```bash
XADD orders * user 1001 amount 99          # 生产
XGROUP CREATE orders g1 0 MKSTREAM        # 建消费组（0=从头消费）
XREADGROUP GROUP g1 c1 COUNT 10 BLOCK 5000 STREAMS orders >   # 消费
XACK orders g1 <message-id>                # 确认
XPENDING orders g1                         # 查未确认的（掉线消费者的活儿）
```

- `>` 表示"还没人领过的"，消费者崩了没 ACK 的消息会进 Pending，用 `XPENDING` + `XCLAIM` 认领。
- 比 List 的 `BRPOPLPUSH` 可靠：有消费组、ACK、Pending 三件套，消息队列场景优先用 Stream。
- 配合 `MAXLEN ~ 10000` 给 stream 封顶，防止无上限膨胀。

## Redis 8 新特性（2026-10-18 按 redis.io 核验）

- **Vector Set（beta）**：原生向量数据类型，做语义检索不用再搭外部向量库，小说引擎 RAG 可选它当轻量向量层。
- **HGETEX / HSETEX / HGETDEL**：hash 字段级"读并设置过期"，缓存/会话场景少一次往返。
- 命令执行快 87%（官方数据）；Community Edition 改名 Redis Open Source。
- 许可注意：Redis 2024 年后一度换 SSPL，2025-05 已改回 AGPL；纯自用/内网无影响，云上托管 Redis 注意合规。
- 来源：https://redis.io/docs/latest/develop/whats-new/8-0/

## Cluster 槽位与扩缩容

- 16384 个槽位按 CRC16(key) % 16384 分配，`{tag}` 花括号内相同才同槽（见疑难杂症 CROSSSLOT）。
- 扩容：`redis-cli --cluster add-node` 加节点 → `reshard` 迁槽 → `rebalance`，槽是逐步搬的，在线不中断。
- 缩容反向操作，先 `reshard` 把槽搬空再 `del-node`，直接下掉节点会丢槽。
- 槽位没覆盖全（`cluster info` 看 `cluster_state:fail`）时整个集群拒绝写，先 `fix` 再查应用。

## Redisson 常用件（限流、信号量）

```java
// 分布式限流：每秒 10 个
RRateLimiter limiter = redisson.getRateLimiter("api:limit");
limiter.trySetRate(RateType.OVERALL, 10, 1, RateIntervalUnit.SECONDS);
limiter.acquire();  // 拿不到就阻塞，可 tryAcquire(0) 非阻塞

// 分布式信号量：最多 5 个并发
RSemaphore s = redisson.getSemaphore("job:slots");
s.trySetPermits(5);
if (s.tryAcquire()) { try { /* 干活 */ } finally { s.release(); } }
```

- 比手写 Lua 省心：Redisson 的 Lua 脚本久经考验，自己写容易漏边界。
- 注意：Redisson 客户端版本要和 Redis Server 版本对上，差太多会有命令不支持。

## 内存分析：MEMORY DOCTOR

```bash
MEMORY STATS                    # 总览：peak、fragmentation ratio
MEMORY USAGE mykey              # 单个 key 占多少
--bigkeys                       # 找大 key（redis-cli --bigkeys）
MEMORY DOCTOR                   # 一句话诊断
```

- `mem_fragmentation_ratio` < 1 说明内存碎片严重（used 比 RSS 小），重启或等碎片整理。
- 大 key 治理：拆（hash 分片）、压（序列化选 msgpack 不用 json）、限（value 别超 10KB）。
