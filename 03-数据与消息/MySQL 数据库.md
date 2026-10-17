# MySQL 数据库

> 每晚 20:30 自动补充完善。项目用 OceanBase MySQL 模式，兼容 MySQL 5.7 语法，绝大多数经验直接通用；OB 特有差异见文末。

## 索引

### 最左前缀原则
```sql
-- 联合索引 (a, b, c)
WHERE a=1 AND b=2          -- 走索引
WHERE b=2 AND c=3          -- 不走（缺 a）
WHERE a=1 AND c=3          -- 走 a 部分，c 不走
```

### 索引失效的 6 个经典场景
1. 对索引列做函数/运算：`WHERE DATE(create_time)='2024-01-01'` → 改范围查询。
2. 隐式类型转换：`varchar` 列用数字查 `WHERE phone=138001` → 索引失效，必须加引号。
3. `LIKE '%xxx'` 前缀模糊：改后缀 `LIKE 'xxx%'` 或上 ES。
4. `OR` 两边有一边没索引。
5. `!=` / `NOT IN`：一般不走索引。
6. 优化器觉得全表更快（小表、过滤度极低）。

### 覆盖索引与回表
- `SELECT` 的列全在索引里 → 覆盖索引，不回表，最快。
- 索引下推（ICP）：5.6+，在索引层先过滤再回表。

## EXPLAIN 速看

```sql
EXPLAIN SELECT * FROM t_order WHERE user_id=1;
```
关键列：
- **type**：`const` > `ref` > `range` > `index` > `ALL`。看到 `ALL` 先警觉。
- **key**：实际用的索引；`NULL` = 没用上。
- **rows**：预估扫描行数，越大越可疑。
- **Extra**：`Using filesort` / `Using temporary` 尽量消掉；`Using index` 是好消息（覆盖索引）。

## 慢查询排查流程

1. 开慢查询日志：`slow_query_log=1`，`long_query_time=1`。
2. `EXPLAIN` 看执行计划，先加索引解决 80% 问题。
3. 索引加了还慢：看是不是**深分页** `LIMIT 1000000,20` → 改游标分页（记住上次 id，`WHERE id>xxx LIMIT 20`）。
4. 再看：大事务、锁等待（`SHOW ENGINE INNODB STATUS`）、连接打满。

## 常见坑

1. **utf8mb4**：建表一律 `utf8mb4`，`utf8` 是假 utf8，emoji 直接报错。
2. **NULL 坑**：`COUNT(col)` 不计 NULL；`WHERE col != 1` 查不出 NULL 的行；能 NOT NULL 就 NOT NULL，给默认值。
3. **隐式转换**：见上，字符串列必须加引号查。
4. **大事务**：事务里别调远程接口、别 sleep，长事务拖死 undo / 锁。
5. **DDL 锁表**：大表加索引用 `ALGORITHM=INPLACE`，业务低峰做；OceanBase 下 DDL 行为有差异，先在测试环境试。
6. **批量插入**：`rewriteBatchedStatements=true`（JDBC URL 参数），性能差一个数量级。
7. **自增 ID**：分布式/分库分表下别依赖自增连续，用雪花算法。

## OceanBase MySQL 模式注意点

- 兼容 MySQL 5.7 协议和语法，JDBC 驱动直接用 `mysql-connector-java`。
- 分布式架构：单分区内事务强一致，**跨分区事务是弱隔离**，写代码时别假设单机 MySQL 的锁行为。
- 自增列行为与单机 MySQL 不同（不保证连续），主键优先雪花 ID。
- 部分 MySQL 系统表/函数不支持，迁移 SQL 先在 OB 测试环境跑一遍。
- 执行计划用 `EXPLAIN` 照常看，但分布式计划多了数据分发算子（EXCHANGE），关注点一样：先看有没有全表扫。

## 待补充
- 分库分表中间件选型笔记
-  binlog / Canal 同步实战
-  OceanBase 租户与资源单元概念
