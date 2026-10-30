---
知识点: Redis 持久化与淘汰策略
层级: 进阶
适用版本: Redis 7.x
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】手动触发快照别用 `SAVE`，用 `BGSAVE`：`SAVE` 会阻塞主进程直到快照写完，实例几十 GB 时整个 Redis 在此期间不响应任何请求；`BGSAVE` 是 fork 子进程在后台写，父进程继续服务。线上只在“确认能停服”的维护窗口用 `SAVE`，日常一律 `BGSAVE`，想确认是否完成看 `INFO persistence` 的 `rdb_last_bgsave_status`。

【小技巧】`INFO persistence` 是持久化排障的第一入口：它集中了 `rdb_last_bgsave_status`、`aof_last_bgrewrite_status`、`aof_current_size`、`aof_last_write_status`，RDB 失败、AOF 重写失败、AOF 写盘失败都在这里留痕。持久化出问题先看这张表，而不是先翻日志，状态字段比日志行更直接。

【小技巧】AOF 重写阈值别乱调，先看默认值：`auto-aof-rewrite-percentage 100`（文件比上次重写后翻倍）且 `auto-aof-rewrite-min-size 64mb`（至少 64MB）时才自动触发。文件长期不重写先确认是不是没到阈值，而不是直接手动 `BGREWRITEAOF`；手动重写后记得看 `aof_last_rewrite_time_sec`，重写本身也会吃 CPU。

【小技巧】Redis 7.0 起 AOF 是多文件结构，别手删单个文件：基础文件 + 增量文件 + manifest 清单文件是一组，`BGREWRITEAOF` 时 manifest 会同时更新。直接删某个 `appendonly.aof.1.incr.aof` 会让 manifest 对不上、启动加载失败；清理 AOF 只走重写和配置，不要像删旧 RDB 那样手动删。

【小技巧】淘汰策略可以运行时改，不用重启：`CONFIG SET maxmemory-policy allkeys-lfu` 立即生效，流量高峰发现 LRU 误伤热点时能先切策略止血、再排查原因。但 `maxmemory` 本身改小要小心，改完瞬间就可能触发一轮淘汰，调小前先看 `INFO memory` 的 `used_memory` 与新上限的差距。

【小技巧】近似 LRU 的采样数可以调：默认 `maxmemory-samples 5`，每次淘汰只采样 5 个 key 取最旧的，样本越小越容易误淘汰热点。热点被误删得频繁时把采样数调到 10 再观察，淘汰精度会上去，代价是每次淘汰多扫几个 key，写多读少的场景不用调它。

【小技巧】LFU 的频率计数会随时间衰减，别拿它当永久热度：`lfu-decay-time` 默认 1 分钟，计数每过这个周期减半，`lfu-log-factor` 默认 10 决定计数长得有多快。昨天的热点今天不再访问，计数会自己衰减下去，这正是 LFU 能抵抗“突发扫描污染”的原因；想让热度留得更久就调大 `lfu-decay-time`，而不是换回 LRU。

【小技巧】写满时读命令不受影响，只有写命令会触发淘汰或报错：淘汰只在写命令让内存超过 `maxmemory` 时才发生，`GET` 再多也不会触发淘汰。`noeviction` 下写满后写命令报 OOM 错误，但 `DEL`、`UNLINK` 这类释放内存的命令照常能用，先删 key 腾空间再恢复写入，不用重启实例。

【小技巧】删大 key 用 `UNLINK` 还不够，清空库用 `FLUSHDB ASYNC`：`FLUSHDB` 默认同步释放，库里有大 key 时一样会卡主进程；`FLUSHDB ASYNC` 把释放交给后台线程，主进程立刻返回。迁移、测试库清理这类“整库不要了”的操作，异步版本是默认选择，同步版本只留给空库或小库。

【小技巧】RDB 文件先校验再拿去恢复：`redis-check-rdb dump.rdb` 能在启动前发现文件损坏，异机备份恢复前跑一遍，比“启动后才发现加载失败”省一轮排障。AOF 同理用 `redis-check-aof` 检查，7.x 多文件 AOF 检查的是 manifest 指向的那组文件，不要只验其中一个。

【小技巧】`MEMORY USAGE` 查大集合时加 `SAMPLES` 控制采样：`MEMORY USAGE key SAMPLES 0` 会全量扫描精确计算，集合几十万元素时很慢；默认采样 5 个元素估算已经够定位问题。只在“要精确数字写进报告”时才用 `SAMPLES 0`，日常定位用默认采样即可。

【小技巧】`save ""` 才是真正关掉 RDB 自动快照的写法：配置文件里删掉所有 `save` 行并不能关快照，默认规则还在；显式写 `save ""` 才能清空自动保存规则。纯缓存实例想彻底不写 RDB，快照规则和 AOF 都要显式关掉，只删配置行等于什么都没关。

【冷知识】RDB 文件头以 `REDIS` 魔数开头：`dump.rdb` 的前 5 个字节就是 ASCII 的 `REDIS`，后面跟版本号数字。拿 `head -c 9 dump.rdb` 就能确认一个文件是不是 RDB、是哪个版本生成的，比靠文件名判断可靠，备份目录里混进别的文件时这个魔数是最快的鉴别方式。

【冷知识】LFU 的频率计数只有 8 位：每个 key 对象头里频率计数占 8 位，最大值 255，计数到顶就不再涨。配合 `lfu-log-factor` 的对数增长，新 key 要攒很久才能追上老热点，这就是“新热点刚进来就被淘汰”的反直觉现象的来源——频率是慢慢攒出来的，不是访问一次就满格。

【冷知识】Redis 的 LRU 时钟只有 24 位，会回绕：对象头里的 LRU 时钟是 24 位毫秒级时间戳，大约 194 天回绕一次。回绕前后比较“谁更旧”要靠带回绕感知的减法，Redis 内部已经处理了，但这也说明 LRU 从一开始就是“够用就好”的近似实现，不是精确的访问时间戳。

【冷知识】AOF 并不是 Redis 一开始就有的：早期 Redis 只有 RDB 快照，AOF 是后来为“丢数据窗口更小”补上的持久化方式。两种机制并存不是设计之初的规划，而是使用场景分化后的结果——这也解释了为什么 7.0 之前 AOF 是单文件、7.0 才改成带 manifest 的多文件结构：AOF 是在 RDB 的地基上长出来的第二套机制。

## 面试与自测

1. 只开 RDB 的缓存实例故障后会丢多少数据？什么场景必须再开 AOF？
2. `allkeys-lru` 下，设了过期时间的 key 会不会被淘汰？想保住不设过期的权威数据该选什么策略？
3. 从节点内存异常时，为什么排查要先看 `replica-ignore-maxmemory`？

### 参考答案

1. 丢最后一次快照之后的数据。纯缓存、可重建的数据可以只开 RDB 甚至都不开，靠集群高可用；Redis 里有不可丢数据（队列、会话）时要开 AOF everysec + RDB 兜底，文件定时备份到异机并实际演练过恢复。
2. 会。`allkeys-*` 的淘汰范围是全部 key，没到期的 key 一样淘汰；`volatile-*` 只在设了过期时间的 key 里淘汰，所以缓存 + 部分权威数据混存时选 `volatile-lru`，前提是权威数据确实不设过期且容量规划留足。
3. 从节点默认忽略 maxmemory（`replica-ignore-maxmemory`），避免从节点自己淘汰导致数据与主不一致；所以从节点内存异常时，先确认这条默认值再往下查，不要按主节点的淘汰逻辑去套从节点。

## 6. 关联知识

- 命令与数据结构基础：[Redis](数据库/Redis/Redis.md)
- 缓存三大问题与锁：[Redis 缓存问题与分布式锁](数据库/Redis/Redis-缓存问题与分布式锁.md)
- 集群下的持久化与复制：[Redis 集群与哨兵](数据库/Redis/Redis-集群.md)
