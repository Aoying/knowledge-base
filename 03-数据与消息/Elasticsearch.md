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
