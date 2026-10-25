---
知识点: Redis 持久化与淘汰策略
层级: 进阶
适用版本: Redis 7.x
更新日期: 2026-10-23
关联:
  - Redis
  - Redis-缓存问题与分布式锁
  - Redis-集群
标签:
  - 数据库
  - 缓存
  - 原理
  - 性能优化
---

# Redis 持久化与淘汰策略

> 《Redis》总篇有 RDB/AOF 的一句话版；本篇展开机制、配置口径和内存问题排查。

## 1. RDB 与 AOF

| 维度 | RDB | AOF |
|---|---|---|
| 原理 | 定时把内存数据生成二进制快照（dump.rdb） | 追加记录每条写命令（appendonly.aof） |
| 触发 | `save 900 1` 等规则 / 手动 `BGSAVE`、`SAVE` | 每条写命令按刷盘策略落盘 |
| 数据风险 | 丢最后一次快照之后的数据 | `everysec` 最多丢约 1 秒 |
| 文件与恢复 | 文件小、恢复快 | 文件大、恢复慢（要重放命令） |
| 开销 | fork 子进程生成，瞬间内存和 CPU 压力 | 持续写盘 IO，命令量大时明显 |

```conf
# redis.conf 常用口径（缓存型实例）
save 900 1
save 300 10
appendonly yes
appendfsync everysec        # always 最安全但最慢；no 靠操作系统刷盘，不可控
```

- **刷盘三档**：`always` 每条命令都 fsync，性能最差；`everysec` 每秒刷一次，是生产默认折中；`no` 交给 OS，丢数据窗口不可控，别用在重要数据。
- **混合持久化（4.0+）**：AOF 重写时，前半段用 RDB 格式存全量、后半段追加增量命令（`aof-use-rdb-preamble yes`），兼顾恢复速度和丢失窗口，7.x 默认开启，生产保持混合模式即可。
- **AOF 重写**：命令越攒越长时用 `BGREWRITEAOF` 压缩（多条命令合并为最终状态）；7.x 改为多文件 AOF（基础文件 + 增量文件 + manifest），原理不变。
- **fork 的代价**：RDB 生成和 AOF 重写都要 fork，靠写时复制；写入越频繁，复制的页越多，极端时内存接近翻倍。大内存实例要给机器留冗余，`vm.overcommit_memory=1` 是官方建议的系统设置。

选型结论：纯缓存、可重建数据 → 只开 RDB 甚至都不开，靠集群高可用；Redis 里有不可丢数据（队列、会话）→ AOF everysec + RDB 兜底，并把文件定时备份到异机。

## 2. maxmemory 与淘汰策略

不设 `maxmemory` 时 Redis 会一直吃内存直到被系统 OOM kill。缓存型实例必设上限（经验口径：不超过机器内存的 60%～70%，给 fork 和系统留量）。

```bash
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

8 种淘汰策略分两组理解：

| 组 | 策略 | 含义 |
|---|---|---|
| 不淘汰 | `noeviction` | 写满后新写入报错，读不受影响（默认） |
| 全键范围 | `allkeys-lru` | 所有 key 里淘汰最近最少使用 |
| | `allkeys-lfu` | 所有 key 里淘汰访问频率最低 |
| | `allkeys-random` | 随机淘汰 |
| 带过期键范围 | `volatile-lru` / `volatile-lfu` / `volatile-random` | 只在设了过期时间的 key 里淘汰 |
| | `volatile-ttl` | 在带过期键里优先淘汰快到期的（剩余 TTL 最短） |

怎么选：

- **纯缓存**：`allkeys-lru` 起步；热点集中、长尾明显的场景换 `allkeys-lfu`（LFU 对突发扫描污染的抵抗更好）。
- **缓存 + 部分权威数据混存**：`volatile-lru`，保证没设过期的 key 不被淘汰——但前提是权威数据确实不设过期，且容量规划留足。
- Redis 用的是**近似 LRU**（采样若干 key 淘汰最旧的），不是精确排序，理解这一点就不用纠结个别 key 的去留。

注意：主从复制时从节点默认忽略 maxmemory（`replica-ignore-maxmemory`），避免从节点自己淘汰导致数据与主不一致，排查从节点内存异常时记得这条。

## 3. 内存碎片

```bash
INFO memory
# used_memory        Redis 逻辑数据占用
# used_memory_rss    操作系统看到的实际占用
# mem_fragmentation_ratio = rss / used，> 1.5 值得关注
```

碎片来源：频繁改写、大量 key 过期后内存没归还 OS。处理手段：开启主动碎片整理 `activedefrag yes`（有少量 CPU 代价）；碎片率长期过高且数据可重建时，低峰重启实例是最干脆的解法。判断标准先看 `MEMORY DOCTOR` 的结论，别只看一个比率就动手。

## 4. 大 key 与热 key（治理篇）

发现问题：

```bash
redis-cli --bigkeys              # 按类型给出最大 key 概览（采样，结果仅供定位）
redis-cli --latency-history      # 延迟历史，辅助判断阻塞
MEMORY USAGE key                 # 单个 key 的实际占用
--hotkeys 需要 maxmemory-policy 为 LFU 才可用（redis-cli --hotkeys）
```

治理口径（和《Redis》总篇大 key 章节一致，这里补落地动作）：

- **大 key**：拆（大 Hash 按字段分片成多个小 Hash）、压（换紧凑序列化）、删用 `UNLINK` 异步释放，禁止对大 key 用 `DEL`、全量 `HGETALL`/`SMEMBERS`。
- **热 key**：应用层本地缓存兜一层（Caffeine）是性价比最高的；或把 key 复制成多份副本随机读。热 key 的根源是业务热点，扩容 Redis 解决不了单 key 热点。
- **预防**：写入侧做约束——单 value 大小上限、集合元素数上限，在代码评审和客户端封装层拦截，别等线上爆了再治理。

## 5. 常见坑

- **只开 AOF 没注意文件膨胀**：重写触发条件（`auto-aof-rewrite-percentage` / `auto-aof-rewrite-min-size`）要保留默认并监控磁盘，AOF 写满磁盘的表现是写入报错而不是自动清理。
- **备份只存本机**：RDB 文件在本机磁盘上，机器没了备份也没了；定时同步到异机/对象存储并实际演练过恢复，才叫有备份。
- **淘汰策略与业务假设冲突**：以为设了过期就不会被淘汰，`allkeys-*` 下没到期的 key 一样淘汰；反过来 `volatile-*` 下没设过期的 key 永不淘汰，内存可能被它们先撑满。

## 6. 关联知识

- 命令与数据结构基础：[Redis](数据库/Redis/Redis.md)
- 缓存三大问题与锁：[Redis 缓存问题与分布式锁](数据库/Redis/Redis-缓存问题与分布式锁.md)
- 集群下的持久化与复制：[Redis 集群与哨兵](数据库/Redis/Redis-集群.md)
