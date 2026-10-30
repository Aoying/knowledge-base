---
知识点: MySQL 索引与执行计划
层级: 进阶
适用版本: MySQL 5.7 / 8.x（OceanBase MySQL 模式通用）
更新日期: 2026-10-31
关联:
  - MySQL-数据库
  - MySQL-事务与MVCC
  - 分库分表
标签:
  - 数据库
  - 索引
  - 原理
  - 性能优化
---

# MySQL 索引与执行计划

> 《MySQL-数据库》里已有最左前缀、失效场景和 EXPLAIN 速查；本篇讲结构原理和执行计划的完整读法，两篇配合看。

## 1. 核心概念

索引的本质是**用额外的存储和写入代价，换查询时少扫行**。InnoDB 用 B+ 树组织索引：查询慢不慢，先看这条 SQL 在树上走了多远、回表了几次。

## 2. B+ 树为什么适合做索引

- **只有叶子节点存数据**，非叶子节点只存键和指针：一页（默认 16KB）能放更多键，树更矮。千万级数据通常 3～4 层，一次点查只需 3～4 次页访问。
- **叶子节点按顺序链表相连**：范围查询、排序沿链表走即可，不用回树上反复跳。
- 对比：B 树每个节点都存数据，扇出小、树更高；哈希索引只支持等值、不支持范围和排序；跳表是 Redis ZSet 的选择，磁盘场景页利用率不如 B+ 树。

### 聚簇索引与二级索引

- InnoDB 的表数据本身就是按主键排的 B+ 树（聚簇索引），叶子存整行。
- 二级索引（普通/唯一/联合）的叶子只存索引列 + 主键值，所以查非索引列要拿主键再查一次聚簇索引，这就是**回表**。
- 推论：主键用自增/有序 ID 时插入是顺序追加；用 UUID 做主键会随机插页、频繁页分裂，写入和缓存都吃亏。

## 3. 联合索引与最左前缀（深化）

联合索引 `(a, b, c)` 的排序是先 a、再 b、再 c，所以：

```sql
-- 索引 idx_abc (a, b, c)
WHERE a = 1 AND b = 2 AND c > 3   -- a、b 等值定位，c 做范围
WHERE a = 1 AND c = 3             -- 只用到 a；c 在 b 不确定时无序
WHERE a > 1 AND b = 2             -- a 是范围，b 用不上（范围列后面的列失效）
ORDER BY a, b                     -- 顺序与索引一致，免排序
```

设计口诀：**等值列在前、范围列在后；区分度高的列优先考虑，但要服务最常用的查询组合**。别给每个查询都建一个索引——索引越多写入越慢，联合索引能覆盖的单个查询就不要再建单列索引（`(a,b)` 已能服务只查 a 的场景）。

### 覆盖索引与索引下推

- **覆盖索引**：查询列全在索引里，不回表。`EXPLAIN` 的 Extra 出现 `Using index`。
- **索引下推（ICP，5.6+）**：二级索引里能先判断的条件（如联合索引后列的部分条件），在存储引擎层先过滤，减少回表次数，Extra 出现 `Using index condition`。

```sql
-- 例：idx_name_age (name, age)，name 前缀匹配 + age 过滤可下推
SELECT * FROM t_user WHERE name LIKE '张%' AND age = 20;
```

## 4. EXPLAIN 完整读法

```sql
EXPLAIN SELECT * FROM t_order WHERE user_id = 1001 AND status = 0;
-- 8.0 可用 EXPLAIN FORMAT=TREE 看树形计划；EXPLAIN ANALYZE 直接给实际耗时（谨慎在生产大查询上用）
```

| 字段 | 怎么看 |
|---|---|
| type | `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`；`index` 是全索引扫描，也不好 |
| key / key_len | 实际选用的索引；key_len 可反推联合索引用到了几列 |
| ref | 索引列与什么比较（常量、列、函数） |
| rows | 预估扫描行数，只是估算，和实际差很大时先 `ANALYZE TABLE` 更新统计信息 |
| filtered | 存储引擎返回后还要过滤掉的比例，越接近 100 越好 |
| Extra | `Using index` 覆盖索引（好）；`Using index condition` 下推；`Using where` 回表后过滤；`Using filesort` 额外排序；`Using temporary` 临时表（group by/distinct 常见） |

排查顺序：先看 type 是不是 `ALL` → 再看 key 是否选错 → 再看 rows 估算是否离谱 → 最后看 Extra 的 filesort/temporary 能否用调整索引顺序消掉。

### 优化器可能选错索引

统计信息过期、数据分布倾斜时优化器会选错。先 `ANALYZE TABLE t;` 刷新统计；仍不对再考虑 `FORCE INDEX(idx_x)` 临时兜底，并记录下来——强制索引是补丁不是常态，数据分布变了它会变成新的坑。

## 5. 慢 SQL 定位流程

```sql
-- 1. 打开慢日志（会话级先验证，再落全局配置）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;          -- 阈值 1 秒起步，按业务收紧
SHOW VARIABLES LIKE 'slow_query_log_file';

-- 2. 聚合慢日志找 Top SQL（mysqldumpslow 按总耗时排序）
-- mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log

-- 3. 对单条 SQL 看计划和实际开销
EXPLAIN FORMAT=TREE SELECT ...;
SHOW WARNINGS;                            -- 看优化器改写后的 SQL
```

定位闭环：慢日志聚合出 Top 指纹 → EXPLAIN 看计划 → 缺索引就补、选错就刷统计 → 改完用同一条 SQL 复测 rows 和耗时 → 上线后观察慢日志里该指纹是否消失。没有复测的优化不算完成。

深分页和 `SELECT *` 等写法问题《MySQL-数据库》的慢查询排查一节已有，不重复；本篇只补一句：`rows` 估算正常但就是慢，优先怀疑回表太多（把查询列收进联合索引做覆盖）和排序临时表落盘。

## 6. 常见坑

- **索引列上做运算/函数**：`WHERE YEAR(create_time) = 2026` 全表扫；改写成范围 `create_time >= '2026-01-01' AND create_time < '2027-01-01'`，或建函数索引（8.0 支持）。
- **联合索引顺序凭感觉**：把范围列放第一位，后面等值列全废。按「等值在前、范围在后」排。
- **低区分度列单独建索引**：如 status 只有 0/1，单独建了优化器也多半不走；它适合做联合索引的后列。
- **索引建了但统计没更新**：大批量导入/删除后 `ANALYZE TABLE` 一次，再看执行计划。
- **长字符串前缀索引**：`KEY idx (name(10))` 能省空间但无法覆盖索引、不能用于排序，权衡后再用。

## 小技巧与冷知识

【小技巧】查联合索引的列顺序别凭记忆，直接问 `INFORMATION_SCHEMA.STATISTICS`：`SELECT INDEX_NAME, SEQ_IN_INDEX, COLUMN_NAME FROM INFORMATION_SCHEMA.STATISTICS WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 't_order' ORDER BY INDEX_NAME, SEQ_IN_INDEX;`。它能直接看到每个索引由哪些列组成、顺序是怎样的。排错时先对一下顺序，再判断最左前缀到底用到了哪几列，比凭印象猜要准，也能发现把 `(a, b)` 记成 `(b, a)` 这类低级错误。

【小技巧】想删一个疑似无用的索引又不敢直接删，可以先把它设为不可见：MySQL 8.0 支持 `ALTER TABLE t_order ALTER INDEX idx_x SET INVISIBLE;`，设为不可见后优化器会忽略它，效果上等同于删掉，但随时能用 `SET VISIBLE` 恢复。观察一段时间，确认慢日志里没有因为缺它而变慢的查询，再真正执行 `DROP INDEX`，这样试错成本比直接删低得多。

【冷知识】主键越宽，所有二级索引都会跟着变胖：InnoDB 二级索引的叶子节点存的是索引列加主键值，主键并不是只影响聚簇索引本身。原文只提了 UUID 主键会导致随机插页，但宽主键还有另一层代价，就是每个二级索引里都要重复存一份很长的主键值，索引变大、写入放大、缓存占用都会跟着涨。很多人只算主键列本身的长度，忽略了它会被复制到每一个二级索引里。

【冷知识】一条查询不一定只用一个索引：优化器在合适的时候会对同一张表做索引合并，EXPLAIN 的 type 会显示 `index_merge`，Extra 里会出现 `Using union` 或 `Using intersect`。这说明 MySQL 会分别用两个索引各查出一批主键，再合并或取交集后回表，而不是像多数人以为的那样一次查询只能选一个索引。看到 `index_merge` 不用慌，它本身不是坏信号，真正要看的是合并后回表的行数是否仍然很大。

## 面试与自测

1. 联合索引 `(a, b, c)` 下，`WHERE a = 1 AND c = 3` 能用到几列？为什么？
2. EXPLAIN 的 Extra 出现 `Using index`、`Using index condition`、`Using filesort` 各代表什么？
3. `rows` 估算正常但查询就是慢，按本篇应优先怀疑什么？怎么处理？

### 参考答案

1. 只用到 a。联合索引的排序是先 a、再 b、再 c，c 在 b 不确定时无序，所以跳过 b 直接用 c 是用不上的；同理 a 是范围时，后面的 b 也用不上——口诀是等值列在前、范围列在后。
2. `Using index` 是覆盖索引，查询列全在索引里，不回表，是好事；`Using index condition` 是索引下推（ICP），二级索引里能先判断的条件在存储引擎层先过滤，减少回表次数；`Using filesort` 是额外排序，需要看能否用调整索引顺序消掉。
3. 优先怀疑回表太多：把查询列收进联合索引做覆盖；以及排序临时表落盘。排查顺序仍是先看 type 是不是 `ALL`、key 是否选错、rows 估算是否离谱，最后看 Extra 的 filesort/temporary。

## 7. 关联知识

- 基础速查与失效场景：[MySQL 数据库](数据库/MySQL/MySQL-数据库.md)
- 事务可见性如何影响索引扫描读到的版本：[MySQL 事务与 MVCC](数据库/MySQL/MySQL-事务与MVCC.md)
- 分片后跨片查询为什么贵：[分库分表](数据库/分库分表.md)
