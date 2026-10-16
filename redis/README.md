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
- Redis Cluster 槽位与扩缩容
- Stream 消费者组实战
- Redisson 常用件（限流、信号量）
