---
知识点: Elasticsearch
层级: 进阶
适用版本: Elasticsearch 7 / 8
更新日期: 2026-10-24
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
