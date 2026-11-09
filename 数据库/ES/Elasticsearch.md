---
知识点: Elasticsearch
层级: 进阶
适用版本: Elasticsearch 7 / 8
更新日期: 2026-11-09
关联:
  - ES-倒排索引与查询
  - 数据库知识点地图
  - MySQL-数据库
标签:
  - 数据库
  - 搜索
  - 索引
  - 实践
---

# Elasticsearch

> 本篇是 ES 的汇总与索引：概念全景、选型、集群运维、写入与聚合实战留在本篇；搜索内核部分（倒排索引、分析器、mapping、Query DSL、相关性打分）已拆为独立篇（见下方索引），正文不再重复。

分布式搜索与分析引擎，基于 Lucene；核心能力是全文检索，靠倒排索引实现毫秒级关键词搜索。

## 核心概念

| ES | 关系型数据库 |
|---|---|
| index（索引） | database（库） |
| document（文档） | row（行） |
| field（字段） | column（列） |
| mapping（映射） | schema（表结构） |
| shard（分片） | 分库分表 |
| replica（副本） | 主从复制 |

- **倒排索引**：传统索引是"文档→词"，倒排是"词→文档列表"，关键词搜索时查表求交集，不用扫全表。结构细节（词典、倒排列表、词频与位置）见子篇。
- **mapping**：定义字段类型；`text` 做分词全文搜索，`keyword` 做精确匹配/聚合/排序。类型选型与动态映射纪律见子篇。

## 子篇索引

- [ES-倒排索引与查询](数据库/ES/ES-倒排索引与查询.md)：倒排表结构（词典+倒排列表）、分析器三段链与中文分词、mapping 要点与字段类型速查、Query DSL（match/term/bool/range 与 query/filter 上下文）、BM25 打分的三个直觉——搜索效果不对、查不到、排序怪，先去这篇。

## 快速上手

```bash
# 写入文档
curl -X PUT "localhost:9200/books/_doc/1" -H 'Content-Type: application/json' -d'
{"title":"Java 编程思想","author":"Bruce Eckel","year":2007}'

# 全文搜索
curl -X GET "localhost:9200/books/_search" -H 'Content-Type: application/json' -d'
{"query":{"match":{"title":"Java"}}}'
```

## 与 MySQL 对比

| 场景 | 选谁 |
|---|---|
| 精确查询、事务、表关联 | MySQL |
| 全文搜索、模糊搜索、搜索高亮 | ES |
| 日志分析、指标聚合 | ES |
| 主数据存储 | MySQL（ES 不适合当主库） |

典型架构：MySQL 存主数据 → 同步到 ES → 搜索走 ES。

## Java 客户端

- 7.x：`RestHighLevelClient`（已废弃，但存量项目多）
- 8.x：官方 `Java API Client`（`elasticsearch-java`），类型安全，新项目用它

## 5 个常见坑

1. **mapping 建错类型改不了**：text/keyword 定错只能重建索引（reindex），上线前想清楚。
2. **text 字段不能聚合排序**：要聚合/排序用 keyword，或开 fielddata（不推荐）。
3. **深度分页性能坑**：`from+size` 翻到几万页会很慢，大数据量翻页用 `search_after`。
4. **写入后查不到**：默认 refresh 间隔 1s，刚写入的文档有延迟可见，不是丢了。
5. **分片数拍脑袋**：分片不是越多越好，每个分片都是开销；单机测试 1 分片起步。

## 疑难杂症

### 1. 脑裂（split-brain）
- **现象**：网络抖动后集群出现两个 master，数据写乱。
- **原因**：master 选举的 quorum 数设小了，网络分区后两边各自选出 master。
- **解决**：`discovery.zen.minimum_master_nodes`（7.x 前）设为 `master 节点数/2+1`；8.x 用集群自动管理；生产 master 节点至少 3 个。

### 2. bulk 写入被拒绝（rejected execution）
- **现象**：批量写入报 `EsRejectedExecutionException`，队列满了。
- **原因**：写入线程池 + 队列满了，通常是 bulk 包太大、并发太高，或集群写入本来就慢（如 refresh 太频繁）。
- **解决**：bulk 单包控制在 5-15MB；客户端做退避重试；写入高峰调大 `refresh_interval`（如 30s）。

### 3. 分片长期 UNASSIGNED
- **现象**：集群状态 yellow/red，有分片一直分配不下去。
- **原因**：`GET _cluster/allocation/explain` 看，大概率是磁盘水位（flood stage）、节点属性不满足，或副本数 > 数据节点数。
- **解决**：先看 explain 的 decision；磁盘问题清数据或扩容；单节点测试环境把副本数设 0。

## ES|QL 入门（8.11+，9.x 主推）

Elastic 官方新查询语言，管道式写法，比 DSL 好读一个量级（2026-10-18 按 Elastic 官方 release notes 核验，9.3 为当前版本）。

```esql
FROM logs-*
| WHERE status >= 400
| STATS count() BY host
| SORT count() DESC
| LIMIT 20
```

- `FROM` → `WHERE` → `STATS` → `SORT` → `LIMIT`，和写 shell 管道一个感觉。
- 支持 `LOOKUP JOIN` 跨索引关联，`TS` 命令查时序数据。
- Kibana Discover 里直接切 ES|QL 模式就能跑，不用记 DSL 的 JSON 嵌套。
- 9.x 另有 BBQ（Better Binary Quantization）让向量检索快 5 倍，做 RAG 语义检索时值得关注。
- 来源：https://www.elastic.co/docs/release-notes/elasticsearch.md

## 索引生命周期 ILM + 分片规划

```json
PUT _ilm/policy/logs-30d {
  "policy": { "phases": {
    "hot":   { "actions": { "rollover": { "max_age": "7d", "max_size": "50gb" } } },
    "warm":  { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
    "delete":{ "min_age": "30d", "actions": { "delete": {} } }
  } }
}
```

- 日志类索引必配 ILM：hot 写 → warm 缩分片省资源 → 30 天删，不配的话磁盘早晚被吃光。
- 分片数拍脑袋公式：单分片 20-50GB，`分片数 = 数据量/30GB`，别一上来就 5 分片（小数据 1 分片最快）。
- 写入慢先看三件套：`refresh_interval` 调大、bulk 批量、副本数写入时暂时设 0。

## 写入调优三板斧

1. **bulk 批量**：单条 index 是灾难，用 `_bulk`，一批 5-15MB。
2. **refresh_interval**：默认 1s，批量导入时调到 `30s` 甚至 `-1`，导完再调回来。
3. **副本**：大批量写入时先把 `number_of_replicas` 设 0，写完再加回去（省一半写入量）。

```json
PUT /logs-2026.10/_settings { "index": { "refresh_interval": "30s", "number_of_replicas": 0 } }
```

- 写入慢 90% 是 refresh 和副本的锅，调完这两个再怀疑人生。
- 监控看 `indexing` 线程池 reject 数，开始 reject 就说明集群写入到顶了。

## 聚合实战：PV/UV 一把梭

```json
GET /logs/_search
{ "size": 0, "aggs": {
  "by_hour": { "date_histogram": { "field": "@timestamp", "calendar_interval": "hour" },
    "aggs": { "uv": { "cardinality": { "field": "user_id" } } } }
} }
```

- `size: 0` 只要聚合不要文档，省 90% 传输。
- UV 用 `cardinality`（HyperLogLog 近似，误差 <5%），别用 terms 全量去重。
- 聚合嵌套别超过 3 层，bucket 爆炸（`too_many_buckets_exception`）就加 `composite` 分页。

## 基础知识补强

### 核心概念与术语

- 心智模型：ES = 分布式文档库 + 搜索引擎；文档是 JSON，进索引时被「分析」成词条建倒排索引，查询就是拿词去倒排表里查文档集合。分析与倒排的细节见 [ES-倒排索引与查询](数据库/ES/ES-倒排索引与查询.md)。
- 集群与节点：一个 cluster 由多个 node 组成；节点分工——master 负责集群元数据（建索引、分片分配），data node 存数据跑查询，coordinating node 只转发请求。单节点时一个进程把活全干了。
- 分片再认识：一个 index 的数据被切成若干 primary shard 分散到各节点，每个主分片可有若干 replica shard 做备份；主分片数建索引时定死（改要重建），副本数随时可改。
- 集群健康三色：green = 主分片和副本全就位；yellow = 主分片齐、副本缺（能读写，但没冗余，单节点集群常年 yellow 是因为副本没地方放，把副本设 0 即转 green）；red = 有主分片缺失，部分数据不可查，先救集群别查业务。

### 基础语法与用法

```bash
# 集群与索引巡检（_cat 系列，看状态最快）
curl "localhost:9200/_cluster/health?pretty"        # 三色状态、分片数
curl "localhost:9200/_cat/indices?v"                 # 各索引文档数、大小、健康
curl "localhost:9200/_cat/shards?v"                  # 分片分布在哪些节点

# 建索引：settings 在这里定，mappings 的字段类型写法见子篇（定错类型要 reindex）
curl -X PUT "localhost:9200/books" -H 'Content-Type: application/json' -d'
{ "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": { "properties": { "title": { "type": "text", "analyzer": "ik_max_word" } } } }'

# 文档 CRUD 补全
curl "localhost:9200/books/_doc/1"                              # 按 ID 取
curl -X POST "localhost:9200/books/_update/1" -H 'Content-Type: application/json' -d'
{"doc": {"year": 2008}}'                                        # 局部更新（整篇 PUT 是覆盖）
curl -X DELETE "localhost:9200/books/_doc/1"                    # 删一篇；删整个索引也是 DELETE，打错名字很危险，先 GET 确认再删
# 带条件查询（bool+filter 的完整写法与各查询子句讲解在子篇）：
curl -X GET "localhost:9200/books/_search" -H 'Content-Type: application/json' -d'
{ "query": { "bool": { "must": { "match": { "title": "Java" } },
    "filter": [ { "term": { "author": "Bruce Eckel" } } ] } },
  "sort": [ { "year": { "order": "desc" } } ], "from": 0, "size": 10,
  "highlight": { "fields": { "title": {} } } }'
```

```bash
# _bulk 批量写法要点：每篇文档两行（动作行 + 数据行），结尾必须有换行
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/x-ndjson' -d'
{"index": {"_index": "books", "_id": "2"}}
{"title": "深入理解计算机系统", "author": "Bryant", "year": 2016}
{"index": {"_index": "books", "_id": "3"}}
{"title": "数据密集型应用系统设计", "author": "Kleppmann", "year": 2017}
'
# 返回里逐条看 items 有没有 error：bulk 是「整包接收、逐条成败」，不能只看 HTTP 200

# 只数数量不取文档：_count；查返回总数被截断时看 hits.total
curl "localhost:9200/books/_count" -H 'Content-Type: application/json' -d'
{"query": {"match": {"title": "系统"}}}'
```

### 关键机制速览

- 近实时（NRT）：文档写入先进内存缓冲，refresh 后（默认 1s）生成可搜索的段，所以是「近」实时；急查刚写的数据可用 `?refresh=wait_for`，但别在高频写入里这么用。
- 段（segment）不可变：Lucene 把每次 refresh 的数据写成一个不可变段文件，查询查所有段的并集；删除只是打标记，后台合并（merge）时才真正清掉——这解释了为什么频繁更新又贵又胀。
- 写入走主分片、查询可打到副本：副本不只是备份，也分担读流量；写放大和读能力都随副本数线性变化。

### 新手易错点

- 把 ES 当主库用：没有真正的多文档事务、更新代价高，主数据放 MySQL，ES 只做可重建的搜索副本（重建靠全量导入或同步链路）。
- size 设巨大一次全取：单次查询默认最多返回 10000 条（max_result_window），要导全量用 scroll / search_after 游标，别调大窗口硬扛。
- 日期字段格式混写：同一字段一会儿 `2026-10-22` 一会儿时间戳，会写入失败或排序错乱，mapping 里把 format 写全、写入端统一格式。
- 单节点副本留 1：副本无节点可放，集群常年 yellow 还吓人一跳；单节点测试环境副本设 0。
- 搜索类易错点（term 查 text 查不到、动态映射裸奔、中文不装分词器）已随查询内容拆到 [ES-倒排索引与查询](数据库/ES/ES-倒排索引与查询.md)，排查搜索问题先看那篇。

## 实战案例（项目沉淀）

- [数据分析平台及统一门户](项目沉淀/数据分析平台及统一门户.md)：ES 在数据分析平台中的使用
- [翼安反诈平台](项目沉淀/翼安反诈平台.md)：ES 检索实战

## 分片与查询定量：分片大小、深翻页与 refresh 算一算

本节只算数，不重复定性做法：所有数字都是估算口径，生产值以实测为准。

### 分片大小：10–50GB 区间怎么换算成分片数

Elastic 官方建议单分片大小落在 10–50GB 之间（日志与时序类数据），目标值常取 20–50GB；同时单分片文档数建议控制在 2 亿条以内。两个上限取先到者：哪个先触顶，哪个说了算。

- 换算公式：`主分片数 ≈ 主数据量 ÷ 目标单分片大小`，结果向上取整；副本不参与这个除法，副本是复制已有分片，不改变主分片的切分粒度。
- 分片太小与太大的代价方向相反：分片太小，Lucene 实例、元数据与文件句柄随分片数线性涨；分片太大，节点故障后恢复要跨网络复制整个分片，恢复时间也随大小线性涨。
- 官方还给了一个数量上限参考：单分片最多约 21.47 亿篇文档（Lucene 的 `MAX_DOC` 限制），但这只是硬上限，2 亿条的经验上限远比它先到。

**算例 1——900GB 日志要切多少主分片：** 按目标 30GB 算，`900 ÷ 30 = 30` 个主分片；按区间两端算，`900 ÷ 50 = 18` 个，`900 ÷ 20 = 45` 个。所以答案不是一个数，而是一个区间：18–45 个，取 30 个是区间中点附近的折中。数据量翻倍到 1.8TB，分片数也翻倍到 36–90 个——分片数与数据量成正比，不是固定值。

**算例 2——文档数上限什么时候先到：** 单篇文档平均 250B 的小文档，`2 亿 × 250B = 50GB`，文档数上限与 50GB 大小上限同时触顶；单篇文档平均 2KB 时，一个 50GB 分片只能装约 2621 万篇文档（`50 × 2^30 ÷ 2048 ≈ 26,214,400`），远不到 2 亿条，大小上限先到。结论：文档越小，越要同时盯文档数；文档越大，大小上限基本说了算。

### refresh：默认 1s 的近实时到底意味着什么

`index.refresh_interval` 默认 1s：写入先进内存缓冲，每 1s 生成一个可搜索的新段，文档最长约 1s 后可见。这就是“近实时”三个字的定量含义——可见延迟在 0–1s 之间抖动，不是 0，也不是分钟级。

- 每天的 refresh 次数：`86,400 ÷ 1 = 86,400` 次；调到 30s 后是 `86,400 ÷ 30 = 2,880` 次，减少为原来的 1/30。refresh 次数少了，段的数量与后续合并压力跟着降，这是调大间隔带来写入收益的算术来源。
- 每个段攒多少文档：写入速率 10,000 篇/秒时，1s 间隔每个段约 10,000 篇，30s 间隔每个段约 300,000 篇（`10,000 × 30`）。段越大、数量越少，查询时要扫的段文件越少，但单个段生成时占用的内存缓冲也越大。
- 调大间隔的代价是可见延迟同步放大：30s 间隔下刚写入的文档最长 30s 后才可见，实时搜索场景不能这么调；批量导入场景可以先调到 30s 甚至 `-1`（关闭），导完再调回 1s。

**算例 3——refresh 从 1s 调到 30s 少了多少次：** 一天少做 `86,400 − 2,880 = 83,520` 次 refresh，约为原来的 30 倍差距。注意这是次数账，不是吞吐账：写入吞吐实际提升多少取决于段合并与 CPU 的瓶颈占比，次数减少 30 倍不等于写入快 30 倍。

### 深翻页：from+size 的代价是每分片取 from+size 再归并

`from+size` 的代价公式是：每个分片都要取出 `from + size` 条文档排好序，协调节点再把各分片的结果归并、排序、丢弃前 `from` 条，只返回 `size` 条。堆内存与耗时与 `from + size` 成正比，与最终返回的 `size` 无关。

- 默认安全线：`index.max_result_window` 默认 10,000，`from + size > 10,000` 直接被拒绝。想翻第 10,001 条之后的数据，调大这个值是硬扛，不是解法。
- `search_after` 的代价结构不同：它拿上一页最后一条的排序值当游标，每分片只需取 `size` 条，不再随页码变深而膨胀；代价是只能顺序向前翻，不能随机跳页，且排序里要带唯一字段做 tiebreaker（如 `_id`），否则同值文档会在页间重复或丢失。

**算例 4——10 个分片翻到第 9,991 条：** `from = 9,990，size = 10` 时，每个分片取 10,000 条，10 个分片共取 `10 × 10,000 = 100,000` 条，归并后只返回 10 条，取回量是返回量的 10,000 倍。同样的位置用 `search_after` 每页每分片只取 10 条，10 个分片共取约 100 条，取回量随页码变深基本不变。翻得越深，两者的倍数差越大。

### bulk：批量大小换算成文档数与请求数

官方调优文档不给一个固定最优值：做法是从 100 篇起步，每次翻倍测速，速度开始走平就是该数据的最优点；同时提醒单请求别超过几十 MB，并发大了会把集群内存压垮。本文沿用原文已有的 5–15MB 经验区间，只做换算。

- 换算公式：`每包文档数 ≈ 批量字节数 ÷ 单篇文档平均大小`；`总请求数 ≈ 总数据量 ÷ 单包大小`。
- 5MB 一包、单篇文档 2KB 时每包约 2,560 篇（`5 × 2^20 ÷ 2048`）；10MB 一包约 5,120 篇；15MB 一包约 7,680 篇。文档越大，每包篇数越少，盯字节数比盯篇数更准。
- 1TB 数据按 10MB 一包算，总请求数约 104,857 个（`1,048,576MB ÷ 10MB`）；按每秒 2 个 bulk 包算，约 14.6 小时才能发完（`104,857 ÷ 2 ÷ 3,600`）。这个时间账说明：大批量导入的耗时主要由包大小与并发决定，不是单条写入的快慢。

- 副本也要算进总账：30 个主分片、1 个副本时集群里共 60 个分片（`30 × (1 + 1)`），磁盘占用约 1.8TB（`900GB × 2`）；副本数从 1 加到 2，总分片变成 90 个、磁盘约 2.7TB。主分片数定的是切分粒度，副本数定的是总存储与总分片数的倍数，两笔账别混在一起算。
- 分片数还要过节点这一关：分片均匀分布时，`每节点分片数 ≈ 总分片数 ÷ 数据节点数`。60 个分片落到 3 个数据节点上，每节点约 20 个分片；落到 6 个节点上，每节点约 10 个。节点数翻倍，单节点分片数减半，但分片总数不变，查询时要协调的分片数也不变。
- 段数与 refresh 的关系可以再算一笔：1s 间隔下一天产生约 86,400 个段（每分片），30s 间隔下约 2,880 个段；段数差 30 倍，后台合并要处理的小段数量也差一个量级。段不是免费的：每个段都有自己的倒排索引与文件句柄，查询要扫的段越多，打开的文件与合并的开销越大。

### search_after 与 PIT：翻页一致性的定量代价

- `search_after` 不用 PIT 时是无状态的：两次请求之间如果发生 refresh，新写入的文档可能插进结果里，页间会出现重复或丢失；要保一致性就开 PIT（point in time），PIT 会按 `keep_alive` 时间保留索引状态，代价是这段时间内段文件不能被合并清理，磁盘占用暂时多一块。
- scroll 不再推荐用于实时翻页：scroll 上下文要一直占着堆内存与段，官方 7.x 起就建议深翻页走 `search_after` + PIT，scroll 只留给 reindex 这类后台批量导出。
- 页大小本身也有账：`size = 10` 与 `size = 100` 的差别是每页每分片取回量的 10 倍；翻 1,000 页时，前者总取回约 `1,000 × 10 × 分片数`，后者是前者的 10 倍。页调大省请求数，但单次请求的内存与归并量同步放大，两个方向都要算。

### bulk 并发：请求数与线程数的乘法

- 并发也要换算：4 个客户端线程、每包 10MB 时，同一时刻在途数据约 40MB（`4 × 10MB`）；8 个线程就是约 80MB。官方提醒“单请求别超几十 MB”，说的是单个请求，并发线程数乘上去之后的总在途量也要盯着，否则内存压力是按线程数放大的。
- 批量导入的总耗时公式：`总耗时 ≈ 总请求数 ÷ (并发线程数 × 每线程每秒包数)`。1TB 数据、10MB 一包、共 104,857 个请求，4 线程、每线程每秒 1 包时约 7.3 小时（`104,857 ÷ 4 ÷ 3,600`）；线程翻倍到 8，理论时间减半到约 3.6 小时——但前提是集群写入没到顶，到了顶再加线程只会收到 429 拒绝。

### 边界条件

- 边界 1：单分片 10–50GB 是日志与时序类数据的经验区间，企业搜索等小文档场景可以用更小的分片；区间不是硬上限，超 50GB 也能跑，只是恢复变慢。
- 边界 2：2 亿条/分片的文档数上限与 10–50GB 大小上限取先到者；两个上限都只是经验值，硬上限是 Lucene 的约 21.47 亿篇文档。
- 边界 3：`from + size` 的代价按分片数放大，分片越多、翻得越深，协调节点的归并量越大；`max_result_window` 默认 10,000 是保护线，调大它只会把内存压力往后推。
- 边界 4：refresh 调大与 bulk 调大都只在批量导入场景划算；在线实时写入与实时搜索场景下，refresh 保持 1s、bulk 保持 5–15MB，改大任何一个都会先伤可见性或内存，再谈收益。

## 小技巧与冷知识

【小技巧】写别名做索引切换，别让业务代码直连真实索引名：先建 `books` 别名指向 `books-v1`，重建索引后用 `_aliases` 动作把别名原子切换到 `books-v2`，查询和写入的客户端一行不用改。reindex 失败时别名还能瞬间切回去，这是“mapping 定错只能重建”那条坑对应的标准解法。

【小技巧】大索引 reindex 别同步等，用 `_reindex?wait_for_completion=false` 扔后台，再用 `GET _tasks/<task_id>` 看进度。任务 ID 记下来贴到运维群里，几百万文档的重建不用守着一个 curl 会话，断网也不会丢进度；小索引同步跑就行，不用走这套。

【小技巧】`search_after` 翻页时排序里一定要带一个唯一字段做 tiebreaker：只按 `year` 这种会重复的字段排序，相同值的文档在两页之间会重复出现或漏掉。标准写法是 `sort: [{ "year": "desc" }, { "_id": "asc" }]`，`_id` 兜底保证每一页的边界唯一，这也是官方文档推荐的翻页姿势。

【小技巧】批量改字段别先查出来再逐条 `_update`，用 `_update_by_query` 一次改完：比如把某批文档的 `status` 从 `draft` 改成 `published`，一条查询加一个 `doc` 片段就够。但要先 `GET _count` 确认命中数，改错条件的代价是整批文档一起错，删改类操作先数数再动手。

【小技巧】`_update` 支持 `doc_as_upsert: true`，文档不存在时直接按 doc 内容新建：同步链路里“MySQL 有、ES 可能没有”的场景，一条 `_update` 既能更新又能补建，不用先 GET 判断存在性再决定调 `_doc` 还是 `_update`，竞态窗口也少了一个。

【小技巧】查集群健康别只看三色，顺手看 `GET _cat/thread_pool?v&h=node_name,name,active,queue,rejected`：`rejected` 列一旦开始涨，说明写入或搜索线程池已经到顶，客户端马上会收到拒绝，比等业务报错早半拍。原文讲 bulk 被拒时提到过队列满，这张表就是日常盯队列的入口。

【小技巧】堆内存超过 32GB 反而更慢，设堆时别贪大：JVM 在堆小于 32GB 时用压缩指针（compressed oops），超过这个阈值指针变大、实际可用内存反而缩水。单节点 64GB 内存的机器，堆设 30GB 左右、剩下的留给 Lucene 的文件系统缓存，官方文档也是这个口径，`GET _cat/nodes?v&h=heap.percent` 能看到当前堆占用。

【小技巧】只读的历史索引跑一次 `forcemerge` 合并到 1 个段，查询变快还省磁盘：冷数据不再写入后，`POST /logs-2025.01/_forcemerge?max_num_segments=1` 把几十个小段合成一个，段越少查询时要扫的文件越少。只对不再写入的索引做，热索引上跑 forcemerge 会把写入和合并的 IO 抢成一团。

【小技巧】`_cat/indices` 支持按列排序找大户：`GET _cat/indices?v&s=store.size:desc` 一眼看到哪个索引最占磁盘，磁盘水位报警时先看这张表再决定删哪个。配合 `GET _cat/indices/<pattern>?v&h=index,docs.count,store.size` 可以只看某类索引，不用在几百个索引里肉眼翻。

【小技巧】时间序列日志别手建索引，用数据流（data stream）加 ILM 一起配：写入只打 `logs-app` 这一个名字，ES 自动按时间滚动出 `logs-app-2026.10.31-000001` 这样的后备索引，到期自动删。比原文的 ILM 策略再省一步——连 rollover 的索引名都不用自己算，日期数学表达式 `<logs-{now/d}>` 也不用写了。

【小技巧】脚本里调 ES 别加 `?pretty`，用 `filter_path` 只要需要的字段：`GET _cluster/health?filter_path=status,number_of_nodes` 返回只有两行，监控脚本解析 JSON 不用扛整页输出。`pretty` 是给人眼看的，`filter_path` 是给程序看的，两个参数别混着用。

【小技巧】建索引先写索引模板（index template），别每个索引手敲 settings：模板里定好 `index_patterns`、`number_of_shards`、mapping 骨架和 ILM 策略，之后按天建的日志索引自动继承，不会再出现“这个索引副本是 1、那个是 0”的手滑。8.x 用可组合模板（component template + index template），改公共部分只改一个组件。

【冷知识】每个分片都是一个完整的 Lucene 索引实例：分片不是“数据的一块切片”那么简单，底层每个主分片和副本分片都各跑一个 Lucene 实例、各自维护倒排索引和段文件。这就是原文“分片不是越多越好”的根本原因——分片数翻倍，Lucene 实例数和文件句柄数也翻倍，集群的固定开销跟着涨。

【冷知识】ES 的前身是 Shay Banon 2004 年写的 Compass，Elasticsearch 本体 2010 年 2 月首次发布：他先是为妻子做食谱搜索写了 Compass，后来把它重构成分布式版本才有了 Elasticsearch。所以 ES 从第一天起就是“先有搜索引擎、再长出分布式”，不是先有数据库再加搜索，这个出身解释了它为什么不适合当主库。

【冷知识】7.x 干掉了多 mapping 类型，8.x 连 `_type` 字段都没了：老教程里的 `PUT /index/_doc` 其实是 7.x 的过渡写法，一个索引只剩 `_doc` 这一种类型；8.x 彻底移除类型概念，看到 `mytype` 之类的写法直接按过期处理。另一个 8.x 的大变化是安全默认开启：首次启动自动生成 elastic 用户密码并开启 TLS，不再是 7.x 那种裸奔就能跑起来的默认状态。

【冷知识】索引名必须全小写，而且不能以 `-`、`_`、`+` 开头：这是 ES 的硬性命名规则，建索引时写了大写会直接报 `InvalidIndexNameException`。`_cat`、`.kibana` 这种下划线开头的索引是系统保留的，自己建索引别往那个方向撞，日期后缀索引用 `logs-2026.10.31` 这种小写加点的写法最稳妥。

## 面试与自测

1. mapping 里 text/keyword 类型定错了为什么改不了？主分片数和副本数在“能不能改”上有什么区别？
2. `from+size` 翻到几万页为什么会很慢？大数据量翻页和导全量应该用什么？
3. 刚写入的文档查不到是不是丢了？近实时的原因是什么，急查时有什么办法、为什么不能高频用？

4. 900GB 日志按单分片目标 30GB 要切多少主分片？`from = 9,990、size = 10` 在 10 个分片上实际取回多少条文档，为什么 `search_after` 更便宜？refresh 从 1s 调到 30s 一天少做多少次 refresh？

### 参考答案

1. 类型定错只能重建索引（reindex），上线前就要想清楚。主分片数建索引时定死，改也要重建；副本数随时可改。
2. `from+size` 深翻页很慢，大数据量翻页用 `search_after`；单次查询默认最多返回 10000 条（max_result_window），要导全量用 scroll / search_after 游标，别调大窗口硬扛。
3. 不是丢了：文档写入先进内存缓冲，refresh 后（默认 1s）才生成可搜索的段，所以是“近”实时。急查刚写的数据可用 `?refresh=wait_for`，但别在高频写入里这么用。

4. 900÷30=30个主分片，20–50GB区间为18–45个。10分片共取回100,000条，只返回10条；search_after每片只取size条，不随页码膨胀。refresh从1s调到30s，一天由86,400次降至2,880次，少83,520次。
