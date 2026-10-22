---
知识点: Redis
层级: 进阶
适用版本: Redis 7
更新日期: 2026-10-22
关联:
  - 数据库知识点地图
  - MySQL-数据库
  - Redis-集群标签:
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

## 基础知识补强

### 核心概念与术语

- 心智模型：Redis 是跑在内存里的键值数据库——key 永远是字符串，value 是带类型的结构（String/Hash/List/Set/ZSet 等）；所有操作先想「这个 key 是什么类型」。
- 为什么快：纯内存读写 + 单线程执行命令（命令按顺序一条条来，单条命令天然原子）+ IO 多路复用。推论：一条慢命令会堵住后面所有命令。
- 数据库编号：单机默认 16 个逻辑库（0~15），用 `SELECT 3` 切换；只是命名空间隔离，不做权限/资源隔离，Cluster 模式只用 0 号库。团队项目一般约定只用一个库 + key 前缀区分业务。
- key 命名规范：`业务:对象:ID` 冒号分层，如 `order:detail:1001`；便于按前缀排查和批量管理，禁止无意义的 `a`、`test1` 混进生产库。
- 过期是惰性 + 定时双机制：key 到期不会立刻删（访问时发现过期才删，后台再定期抽查清理），所以「已过期」的 key 短时间还占内存是正常现象。
- 序列化心智：value 最终都是字节，存对象要选定一种序列化方式全队统一，中途换格式老数据会全部反序列化失败。

### 基础语法与用法

```bash
# 连接与自检
redis-cli -h 127.0.0.1 -p 6379 -a ******   # 带密码连接
PING                        # 回 PONG 说明通了
INFO                        # 服务器全景：内存、连接数、命中率；INFO memory 只看内存段
DBSIZE                      # 当前库 key 数量
CONFIG GET maxmemory        # 看配置（改配置用 CONFIG SET，但重启失效，要落配置文件）

# 通用 key 命令（五大类型都适用）
EXISTS order:detail:1001        # 存在返回 1
TYPE order:detail:1001          # 看类型：string/hash/list/set/zset/none
EXPIRE pay:lock:1001 60         # 60 秒后过期；PEXPIRE 用毫秒
PERSIST pay:lock:1001           # 取消过期，变永久
RENAME oldkey newkey            # 改名（目标存在会被覆盖，慎用）
DEL key1 key2                   # 返回实际删掉的个数

# Hash 补全（对象字段）
HSET user:1 name "张三" age 18
HGET user:1 name
HMGET user:1 name age
HINCRBY user:1 age 1
HDEL user:1 age
HEXISTS user:1 name

# List（双端队列，LPUSH+RPOP 就是队列）
LPUSH queue:task t1 t2      # 从左塞入
RPOP queue:task             # 从右取出（FIFO 队列）；BLPOP 是阻塞版取不到就等
LRANGE list 0 -1            # 看全部（0 到 -1）；LTRIM 可裁掉多余部分
LLEN queue:task             # 长度，做队列积压监控

# Set（无序不重复）
SADD tag:java u1 u2 u3
SADD tag:python u2 u4
SISMEMBER tag:java u2       # 是否在集合内（权限/黑名单判断）
SINTER tag:java tag:python  # 交集：共同成员
SUNION / SDIFF              # 并集 / 差集
SMEMBERS tag:java           # 全量成员——大集合禁用，改 SSCAN 分批

# String / ZSet 补全（基础高频）
APPEND counter_log "x"          # 末尾追加，不存在当新建
STRLEN order:detail:1001        # value 长度，排查大 value 用
SET session:abc token EX 1800   # 登录会话经典写法：半小时过期
ZINCRBY rank 10 "u1"            # 分数 +10（游戏积分、热度榜）
ZRANGE rank 0 9 WITHSCORES      # 正序前 10；榜单展示常用
ZRANK rank "u1"                 # 查名次（从 0 开始），查不到返回 nil
ZREM rank "u1"                  # 移除成员
ZCARD rank                      # 成员总数

# 数据选型速决（先问访问模式，再选结构）
# 只存一个值/计数     → String
# 一个对象多个字段    → Hash（别把对象 JSON 塞 String 又想改单字段）
# 队列/最近 N 条      → List
# 去重/成员判断/交集  → Set
# 排序/榜单/按分取范围 → ZSet
# 要交集并集就别用 Hash 硬凑，用 Set 的 SINTER/SUNION 原生命令。

# 慢命令自查
SLOWLOG GET 10              # 最近 10 条慢命令（默认阈值 10ms，slowlog-log-slower-than）
CLIENT LIST                 # 当前连接一览：谁在连、各连了多久
OBJECT ENCODING mykey       # 看底层编码，同一结构数据形态不同编码不同（排查内存异常用）

# 事务与发布订阅（基础了解）
MULTI                       # 开事务，后续命令入队不执行
SET a 1
INCR a
EXEC                        # 一次性顺序执行；中途某条出错其余照常执行（不是关系型事务，不回滚）
SUBSCRIBE ch:notify         # 订阅频道
PUBLISH ch:notify "hello"   # 发布（发后即忘，没订阅就丢，不做可靠消息用）
```

### 关键机制速览

- 单命令原子 ≠ 组合操作原子：`GET` 后判断再 `SET` 两步之间别人能插进来，组合逻辑要靠 Lua 脚本一次发送、整体执行。
- 命中率看 `INFO stats` 的 keyspace_hits /（hits + misses）：长期低于 90% 说明缓存策略或容量有问题，先查再扩容。
- 主从复制是异步的：主机写完立刻返回，从机稍后同步，所以「写完马上读从机可能读不到」是设计如此不是 bug。
- 内存上限与淘汰是一对：只设过期不设 maxmemory，冷数据堆积照样能打满内存；两个一起设。
- value 大小经验线：单 value 控制在 10KB 以内、集合元素数控制在万级以内，超过就拆 key 或换存储。

### 新手易错点

- 把 Redis 当主数据库：关了持久化、又没备份，重启/宕机数据就没；重要数据必须有权威来源（MySQL），Redis 里只放可重建的副本。
- FLUSHDB / FLUSHALL 清空数据：危险命令，FLUSHALL 连其他逻辑库一起清；生产环境建议改名或禁用（rename-command），执行前三确认。
- key 不设过期：缓存、验证码、临时锁都必须带过期时间，否则内存只涨不降，最后靠淘汰误删业务 key。
- 用一个大 Hash/大 Set 装全量数据：元素越多操作越慢且无法按元素过期，拆成多个小 key。
- 循环里逐条发命令：网络往返比命令本身贵得多，批量取用 MGET、批量写用 pipeline，别在 for 里一条条 GET。
- 计数器 INCR 后忘了它对不存在的 key 从 0 开始：`INCR newkey` 直接得 1 并生成永久 key，做限流计数时记得补 EXPIRE。
- 在 Cluster 下还按单机习惯用 SELECT 切库、多 key 命令不带 hash tag：Cluster 只有 0 号库，多 key 操作要同槽（花括号 hash tag）。

## 实战案例（项目沉淀）

- [神农-e贷-法人版](项目沉淀/神农-e贷-法人版.md)：Redis 在贷款平台中的使用
