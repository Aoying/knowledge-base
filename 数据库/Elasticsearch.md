---
知识点: Elasticsearch
层级: 进阶
适用版本: Elasticsearch 7 / 8
更新日期: 2026-10-22
关联:
  - 数据库知识点地图
  - MySQL-数据库
标签:
  - 数据库
  - 搜索
  - 索引
  - 实践
---

# Elasticsearch

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

- **倒排索引**：传统索引是"文档→词"，倒排是"词→文档列表"，关键词搜索时查表求交集，不用扫全表。
- **mapping**：定义字段类型；`text` 做分词全文搜索，`keyword` 做精确匹配/聚合/排序。

## 快速上手

```bash
# 写入文档
curl -X PUT "localhost:9200/books/_doc/1" -H 'Content-Type: application/json' -d'
{"title":"Java 编程思想","author":"Bruce Eckel","year":2007}'

# 全文搜索
curl -X GET "localhost:9200/books/_search" -H 'Content-Type: application/json' -d'
{"query":{"match":{"title":"Java"}}}'
```

## Query DSL 常用

- `match`：全文搜索（先分词再匹配）
- `term`：精确匹配（不分词，查 keyword 字段用它）
- `bool`：组合查询（must/should/must_not/filter）
- `range`：范围，如 `{"range":{"year":{"gte":2000}}}`
- `aggs`：聚合，相当于 GROUP BY / AVG / MAX

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
