---
知识点: SQL调优实战
层级: 进阶
适用版本: 通用
更新日期: 2026-10-31
关联:
  - MySQL-索引与执行计划
  - MySQL-数据库
标签:
  - 数据库
  - 索引
  - 性能优化
  - 实践
---

# SQL调优实战

本篇讲“已经知道要调优之后怎么做”。基础部分不重复：EXPLAIN 每一列的基础读法见 [MySQL-索引与执行计划](数据库/MySQL/MySQL-索引与执行计划.md)第 4 节，联合索引与最左前缀见该篇第 3 节，join 算法（NLJ/BNL/Hash Join）的原理见 [MySQL-数据库](数据库/MySQL/MySQL-数据库.md)执行计划进阶一节。

## 1. 执行计划精读：看组合，不看单列

单列好看不等于计划好，要看几列联动：

- **type 梯队**：system > const > eq_ref > ref > range > index > ALL。目标至少 range；大表上出现 ALL 就是全表扫描，先处理它。但 type 只是访问方式，不是全部——range 扫 90% 的行也不比 ALL 强多少。
- **rows 是估算，不是实测**：优化器按统计信息估的行数。rows 小但实际慢，优先怀疑统计信息过期（见第 6 节）或 Extra 里有 filesort/temporary。
- **filtered**：按条件过滤后剩下的比例估计。filtered 很低（如 10%）说明索引定位到的行里大部分又被 WHERE 滤掉——索引选得不够贴切，考虑联合索引把过滤列也放进去。
- **Extra 组合读法**：
  - `Using index`：覆盖索引，好事。
  - `Using where`：回表后或索引上再过滤，中性，看 rows。
  - `Using index condition`：索引下推生效，好事。
  - `Using filesort`：要额外排序，见第 5 节。
  - `Using temporary`：要临时表，见第 5 节。
  - `Using join buffer`：被驱动表没走索引，见第 3 节。

精读习惯：先看 type 有没有 ALL，再看 Extra 有没有 filesort/temporary，最后看 rows 和 filtered 是否合理。三处都干净，计划基本可用。

## 2. 索引失效场景清单

以下场景索引建了也用不上，排查时按清单逐项对：

- **函数或表达式作用于列**：`WHERE DATE(created_at) = '2026-10-01'` 会让 created_at 上的索引失效。改写方向：把条件改成范围，`WHERE created_at >= '2026-10-01' AND created_at < '2026-10-02'`。
- **隐式类型转换**：字符串列传数字，如 `WHERE phone = 13800000000`（phone 是 varchar），优化器会给列套转换函数，索引失效。改写方向：传参与列类型一致，`WHERE phone = '13800000000'`。
- **OR 条件非全部有索引**：`WHERE a = 1 OR b = 2`，只要有一侧列无索引，整条就可能走全表。改写方向：给两侧都建索引，或拆成 UNION。
- **`!=` / `NOT IN`**：一般走不了索引（个别情况下优化器能转 range，但不要依赖）。改写方向：看业务能不能改成范围或 IN 枚举。
- **LIKE 前导 `%`**：`LIKE '%张'` 用不上索引；`LIKE '张%'` 可以。业务上确实要全文模糊匹配时，考虑全文索引或 ES（见搜索相关篇），不要硬扛 B+ 树。
- **联合索引跳列**：基础规则见索引篇第 3 节（最左前缀）。实战里常见的是查询条件里漏了第一列，却指望第二列的索引生效——不会生效。
- **ORDER BY 与索引顺序不一致**：`ORDER BY a DESC, b ASC`，而索引是 `(a ASC, b ASC)`，排序方向混用就可能触发 filesort。改写方向：调整索引列顺序，或统一排序方向。

## 3. join 优化

join 慢先看两件事：驱动表选得对不对、被驱动表有没有索引。

- **小表驱动大表是直觉，不是定律**：Nested Loop 下，驱动表扫一遍、被驱动表每次查索引，总代价约等于驱动表行数 × 被驱动表单次代价。驱动表越小越好；但如果被驱动表没有可用索引，优化器可能改走 BNL/Hash Join，这时驱动表大小的影响就变了——以 EXPLAIN 为准，不要背口诀。
- **被驱动表必须有索引**：join 键上没有索引时，每行驱动表数据都要全表扫被驱动表一次。改写方向：给被驱动表的 join 键建索引；如果 join 键是函数结果或类型不一致，先修类型与表达式（见第 2 节）。
- **join_buffer 不是越大越好**：BNL 下 join_buffer 放驱动表数据，调大能减少被驱动表扫描次数，但内存占用跟着涨，且治标不治本——优先把被驱动表索引补上。
- **ON 条件与 WHERE 条件分工**：ON 写连接关系，WHERE 写过滤。内连接里两者效果常一样，但外连接里把过滤写进 WHERE 会把外连接变内连接（NULL 行被滤掉），语义就变了。

## 4. 深翻页优化

`LIMIT 100000, 20` 的问题不在 20，而在 100000：数据库要扫描并丢弃前 100000 行，越往后翻越慢。

- **游标分页（推荐）**：记住上一页最后一条的排序键，下一页用它做起点。示例：

```sql
-- 改写前（示例）：越往后越慢
SELECT * FROM orders WHERE status = 'PAID' ORDER BY id LIMIT 100000, 20;
-- 改写后（示例）：每次只定位 20 行
SELECT * FROM orders WHERE status = 'PAID' AND id > 100000 ORDER BY id LIMIT 20;
```

适用边界：游标要求排序键稳定唯一（通常用主键或“业务时间 + id”收尾）；需要跳页（直接跳第 500 页）的场景不适用，跳页需求大时考虑搜索层或限制最大页码。

- **延迟关联**：先在覆盖索引上查出 id，再 join 回表取全字段。示例：

```sql
-- 改写前（示例）：排序 + 回表一起做
SELECT * FROM orders WHERE status = 'PAID' ORDER BY created_at LIMIT 100000, 20;
-- 改写后（示例）：先取 id，再回表
SELECT o.* FROM orders o
JOIN (SELECT id FROM orders WHERE status = 'PAID' ORDER BY created_at LIMIT 100000, 20) t
  ON o.id = t.id;
```

注意：延迟关联省的是回表成本，offset 本身的扫描还在；和游标分页比，它是次选。

## 5. 排序与临时表优化

- **Using filesort 的触发**：排序键没有可用索引；排序方向与索引不一致（见第 2 节）；排序列上有函数。消除方向：给排序键建索引（注意与 WHERE 条件列组成联合索引时，顺序要同时照顾过滤与排序）；排序量实在大时，考虑业务上限制排序范围。
- **Using temporary 的触发**：GROUP BY、DISTINCT、UNION 在没有合适索引时建临时表。消除方向：GROUP BY 的列建索引（分组顺序与索引顺序一致时可免临时表）；能用 UNION ALL 就不用 UNION（UNION 要去重，多一次临时表）。
- **排序与分页叠加时**：先保证 WHERE 的索引，再看 ORDER BY 的索引；两者冲突时（一个查询里过滤列和排序列各需要一个索引），优先保证过滤列，排序放内存里做——小结果集排序不贵，大结果集排序才贵。

## 6. worked 优化案例

以下案例只给改写前后的 SQL 对照与思路，不含真实执行时间；上线前用 EXPLAIN 复核，以官方文档为准。

**案例一：函数致索引失效**

```sql
-- 改写前（示例）：DATE() 让 created_at 索引失效
SELECT * FROM orders WHERE DATE(created_at) = '2026-10-01';
-- 改写后（示例）：范围条件走索引
SELECT * FROM orders WHERE created_at >= '2026-10-01' AND created_at < '2026-10-02';
```

思路：把“对列做运算”改成“对常量做运算”，让索引列保持裸列。

**案例二：深翻页**

```sql
-- 改写前（示例）：offset 越大越慢
SELECT * FROM orders WHERE status = 'PAID' ORDER BY id LIMIT 100000, 20;
-- 改写后（示例）：游标定位
SELECT * FROM orders WHERE status = 'PAID' AND id > 100000 ORDER BY id LIMIT 20;
```

思路：用上一页末尾 id 做起点，把“跳过 N 行”变成“定位到某行”。

**案例三：join 无索引**

```sql
-- 改写前（示例）：被驱动表 order_items 的 order_id 无索引，EXPLAIN 出现 Using join buffer
SELECT * FROM orders o JOIN order_items i ON i.order_id = o.id WHERE o.status = 'PAID';
-- 改写方向（示例）：给被驱动表 join 键建索引，再用 EXPLAIN 复核 type 是否从 ALL/BNL 变成 ref
-- CREATE INDEX idx_items_order ON order_items(order_id);
```

思路：先补索引再谈其他优化；索引补上后仍慢，再看驱动表选择与过滤条件。

## 7. 常见误区

- 索引越多越好：每个索引都是写放大和空间成本，写多的表上滥建索引会拖慢写入；联合索引能覆盖的场景不要拆成多个单列索引。
- EXPLAIN rows 小就一定快：rows 是估算，统计信息过期时估算会失真；同时看 Extra 和实际数据分布。
- 强制索引（FORCE INDEX）解决一切：优化器选错索引时，先查统计信息是否过期、查询写法是否导致失效；FORCE INDEX 是兜底，不是常规手段。
- 分页 offset 小就没事：offset 小只是当前不慢；数据量涨上去后同一条 SQL 会变慢，分页写法要在上线前就按游标或延迟关联写好。

## 小技巧与冷知识

【小技巧】EXPLAIN 只给估算，想看实测用 `EXPLAIN ANALYZE`：MySQL 8.0.18 起支持 `EXPLAIN ANALYZE SELECT ...`，它真的执行一遍查询并输出每一步的实际行数和耗时。rows 估算和实测差很多时，先看 ANALYZE 的实际行数再决定是索引问题还是统计信息问题，别只盯着 EXPLAIN 的估算值猜。

【小技巧】rows 估算失真时先跑 `ANALYZE TABLE`，别急着加索引：优化器按表统计信息估行数，数据大批量导入、删除后统计信息会过期，`ANALYZE TABLE orders` 重新采样后，同一条 SQL 的执行计划可能直接变好。它只读表做采样、不锁表写数据，是排查“昨天还快、今天突然慢”的第一步。

【小技巧】优化器选错索引时，先用 optimizer trace 看它为什么这么选：`SET optimizer_trace="enabled=on"` 后跑一遍查询，再查 `INFORMATION_SCHEMA.OPTIMIZER_TRACE`，里面会列出每个候选索引的估算代价和被否决的原因。比 FORCE INDEX 更治本——看到“代价算错是因为统计信息偏了”，就知道该 ANALYZE 还是该改写。

【小技巧】调优先定位，别凭感觉：开慢查询日志（`slow_query_log=1` + `long_query_time` 设阈值）拿到“真实慢的 SQL + 出现频次”，再按频次排序逐条调。线上最慢的往往不是你以为的那条；没有慢日志兜底的调优，很容易把时间花在一年跑一次的查询上。

【小技巧】`SELECT *` 会毁掉覆盖索引：即使 WHERE 的列都有索引，`SELECT *` 也要回表取所有字段，覆盖索引省下的回表成本就没了。只取需要的列，查询才能真正走 `Using index`；字段实在多时，先确认“这些字段是不是这次真的都要”，而不是图省事写星号。

【小技巧】长字符串列建索引前先算区分度：`SELECT COUNT(DISTINCT LEFT(url, 20)) / COUNT(*) FROM t`，比值越接近 1 区分度越好，前缀长度就取到“区分度刚够用”为止。前缀太短区分度不够、优化器不爱用；太长又和全列索引差不多大，省不下空间。

【小技巧】`IN` 列表太长时优化器可能直接放弃索引：几百上千个值的 `IN (...)`，优化器估算成本后可能转成全表扫描。超过几十个值就考虑先把值写进临时表再 JOIN，或者分批查；`IN` 是给“少量枚举”准备的，不是批量传参的通道。

【小技巧】join 之前先把驱动表滤小：WHERE 里的过滤条件如果只和驱动表有关，优化器通常会先过滤再 join，但写成子查询或派生表时过滤可能被推迟。join 慢时先确认驱动表进 join 之前已经被过滤到最小行数——驱动表少一行，被驱动表就少查一次索引。

【小技巧】游标分页的排序键用“业务时间 + 主键”组合：单用 `created_at` 做游标键时，同一秒多条记录会让边界重复或漏行；写成 `WHERE (created_at, id) > ('2026-10-01 10:00:00', 12345)`，时间相同再按 id 兜底，翻页边界才稳定。MySQL 支持行构造器比较，不用拆成两段 OR。

【小技巧】`SHOW PROFILE` 已废弃，别再用它定位耗时：MySQL 8.0 里 `SHOW PROFILE` 是 deprecated 状态，替代品是 Performance Schema 的 `events_statements_history_long` 和 `events_stages_history_long`，能按阶段（解析、加锁、排序、发包）拆耗时。老教程里的 `SET profiling=1` 在新版本上迟早跑不动，排障脚本直接写 Performance Schema 的查询。

【冷知识】`Using filesort` 的名字是历史遗留，不一定真的写文件：它指的是“不在索引顺序里、需要额外排序”，排序量小于 `sort_buffer_size` 时在内存里排，只有超了才落盘。看到 filesort 先看排序的数据量，小结果集的 filesort 成本很低，不值得为它单独加索引。

【冷知识】Hash Join 是 MySQL 8.0.18 才引入的：8.0.18 之前没有等值连接的 Hash Join，被驱动表无索引时只能走 Block Nested Loop；8.0.18 起优化器可以在无索引时选 Hash Join，8.0.20 起 BNL 退居次要位置。原文说“被驱动表无索引时优化器可能改走 BNL/Hash Join”，这个“可能”是有版本边界的，读老版本文档时别把 Hash Join 当默认选项。

【冷知识】MySQL 8.0.13 引入了跳跃扫描（skip scan）：联合索引 `(a, b)` 上查 `WHERE b = 2`（漏了第一列 a）时，优化器可以对 a 的每个不同值各做一次范围扫描，相当于把“跳列失效”部分救回来。但它只在 a 的不同值很少时才划算，不同值一多代价立刻超过全表扫描，所以它是个“有条件生效”的优化，不是“跳列从此没事了”。

【冷知识】索引下推（ICP）和多范围读（MRR）都是可以开关的优化器特性：`optimizer_switch` 里 `index_condition_pushdown`、`mrr`、`mrr_cost_based` 这几个开关控制着它们是否启用。EXPLAIN 里看到 `Using index condition` 就是 ICP 在工作；某个查询怀疑是 MRR 导致变慢时，可以会话级关掉对应开关做 A/B 对照，而不是直接改 SQL。

【冷知识】`EXPLAIN` 的 `filtered` 是“按百分比估算的剩余比例”，不是实测比例：它由统计信息算出，数据倾斜严重时（比如 status='PAID' 占 99%），filtered 的估算会和实际差一个量级。filtered 很低但实际跑得快，或者反过来，都先怀疑统计信息和数据分布，别只看这一个数字下结论。

## 面试与自测

1. `WHERE DATE(created_at) = '2026-10-01'` 为什么用不上索引？怎么改？
2. `LIMIT 100000, 20` 慢的根本原因是什么？游标分页的适用边界是什么？
3. join 慢时先看哪两件事？被驱动表没有索引时优化器一般会怎么走？

### 参考答案

1. 因为函数作用于列，优化器无法直接用 B+ 树定位，只能逐行算 DATE() 再比较，索引失效。改成范围条件：`WHERE created_at >= '2026-10-01' AND created_at < '2026-10-02'`，让索引列保持裸列。
2. 根本原因是数据库要扫描并丢弃前 100000 行，越往后翻成本越高。游标分页用上一页末尾的排序键做起点（`WHERE id > 上次 max ORDER BY id LIMIT n`），每次只定位 n 行；边界是排序键必须稳定唯一，且不支持跳页，跳页需求大时考虑搜索层或限制最大页码。
3. 先看驱动表选得对不对、被驱动表有没有索引。被驱动表无可用索引时，优化器可能改走 BNL 或 Hash Join；这时驱动表大小的影响就变了，要以 EXPLAIN 为准，不要只背“小表驱动大表”。

## 关联知识

- [MySQL-索引与执行计划](数据库/MySQL/MySQL-索引与执行计划.md)：EXPLAIN 基础读法、联合索引与最左前缀、覆盖索引与索引下推。
- [MySQL-数据库](数据库/MySQL/MySQL-数据库.md)：执行计划进阶（join 算法）、慢查询排查速记。
- [锁等待与死锁分析](数据库/MySQL/锁等待与死锁分析.md)：SQL 慢的原因如果是锁等待，按该篇流程排查。
- [慢查询诊断](数据库/MySQL/慢查询诊断.md)：从慢日志到上线验证的完整闭环。
