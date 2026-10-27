---
知识点: SQL基础与窗口函数
层级: 基础
适用版本: MySQL 8.0+
更新日期: 2026-10-27
关联:
  - MySQL-数据库
  - MySQL-索引与执行计划
  - SQL调优实战
标签:
  - 数据库
  - 实践
---

# SQL基础与窗口函数

> 分工说明：本篇只管“怎么写对”——语法、语义、易错点。写得对之后怎么变快（索引、执行计划、慢查询闭环）见专题篇，不在本篇重复。DML 四件套、JOIN 三连、聚合分组的速记版在总纲篇，[MySQL 数据库](数据库/MySQL/MySQL-数据库.md)；本篇是在速记版之上的完整讲法与边界情况。

示例表（虚构电商库 shop，全篇共用）：

```sql
CREATE TABLE t_user (
  id   BIGINT PRIMARY KEY,
  name VARCHAR(64) NOT NULL,
  city VARCHAR(32) NOT NULL
);

CREATE TABLE t_order (
  id          BIGINT PRIMARY KEY,
  user_id     BIGINT NOT NULL,
  order_no    VARCHAR(32) NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,
  status      TINYINT NOT NULL DEFAULT 0,
  create_time DATETIME NOT NULL
);

CREATE TABLE t_order_item (
  id           BIGINT PRIMARY KEY,
  order_id     BIGINT NOT NULL,
  product_name VARCHAR(64) NOT NULL,
  qty          INT NOT NULL,
  price        DECIMAL(10,2) NOT NULL
);
```

## 1. 核心概念

- SQL 分四类：DML（INSERT/UPDATE/DELETE/SELECT，改数据查数据）、DDL（CREATE/ALTER/DROP，改结构）、DCL（GRANT/REVOKE，管权限）、TCL（事务，START TRANSACTION/COMMIT/ROLLBACK）。
- 本篇的“写对”有三个检验标准：结果集对（行、列、值都没多没少）、语义稳（换数据量、换边界值结果依然对）、可维护（别人接手能看懂意图）。
- 一条 SQL 的语义由逻辑执行顺序决定，不是由书写顺序决定——第 7 节是全篇的地基。

## 2. DML 基础

### INSERT：三种形态

```sql
-- 单行
INSERT INTO t_order (user_id, order_no, amount, create_time)
VALUES (1001, 'NO2026001', 99.00, NOW());

-- 多行一次写（比循环单条快得多）
INSERT INTO t_order (user_id, order_no, amount, create_time) VALUES
  (1001, 'NO2026002', 59.00, NOW()),
  (1002, 'NO2026003', 199.00, NOW());

-- INSERT ... SELECT：从查询结果批量导数
INSERT INTO t_order_archive (id, user_id, amount)
SELECT id, user_id, amount FROM t_order WHERE create_time < '2025-01-01';

-- 存在则更新（order_no 有唯一键时才有效）：幂等写入的常用写法
-- 8.0.19+ 用行别名写法，老 VALUES() 函数已废弃
INSERT INTO t_order (user_id, order_no, amount, create_time)
VALUES (1001, 'NO2026001', 99.00, NOW()) AS new
ON DUPLICATE KEY UPDATE amount = new.amount;
```

### UPDATE / DELETE：先 SELECT，再执行

```sql
-- 危险写法：忘 WHERE = 全表遭殃（总纲易错点第 1 条）
UPDATE t_order SET status = 1;              -- 别这样写

-- 安全写法：先用相同 WHERE 跑 SELECT 确认行数，再执行
SELECT COUNT(*) FROM t_order WHERE id = 10 AND status = 0;
UPDATE t_order SET status = 1 WHERE id = 10 AND status = 0;

-- 批量改：用主键或唯一键分批，避免一次锁太多行
UPDATE t_order SET status = 2 WHERE status = 1 AND id BETWEEN 1000 AND 1999;
DELETE FROM t_order WHERE status = 9 AND create_time < '2024-01-01' LIMIT 1000;
```

### SELECT 基本形态

```sql
SELECT id, order_no, amount
FROM t_order
WHERE user_id = 1001 AND status = 1
ORDER BY create_time DESC
LIMIT 20;
```

- 按需列字段，别用 `SELECT *`：多取无用列、表结构一变调用方就可能炸，覆盖索引也失效。
- `LIMIT n` 不带 `ORDER BY` 时返回顺序不保证：要“最新 20 条”必须显式 `ORDER BY`。

## 3. JOIN 类型与选择

### 五种 JOIN，一张决策表

| 类型 | 语义 | 典型用途 |
|---|---|---|
| INNER JOIN | 两边都匹配才出 | 取“有关联”的数据，如已下单用户 |
| LEFT JOIN | 左表全出，右表缺失补 NULL | 取“全部左表 + 尽量补右表”，如全部用户及其订单 |
| RIGHT JOIN | 右表全出 | 少用，一般改写成 LEFT JOIN（把表换边） |
| CROSS JOIN | 笛卡尔积 | 极少直接用，常见于“每行配全部维度”的构造查询 |
| 自连接 | 自己连自己 | 查层级、查“同一表内两行的关系”，如员工与上级 |

```sql
-- 内连接：只要有订单的用户
SELECT DISTINCT u.name
FROM t_user u
INNER JOIN t_order o ON o.user_id = u.id;

-- 左连接：全部用户，没有订单的也出（订单字段为 NULL）
SELECT u.name, o.order_no
FROM t_user u
LEFT JOIN t_order o ON o.user_id = u.id;

-- 自连接：同一订单下的商品两两配对（找“一起买过”的组合）
SELECT a.product_name, b.product_name
FROM t_order_item a
JOIN t_order_item b
  ON a.order_id = b.order_id AND a.id < b.id;
```

### MySQL 没有 FULL JOIN，用 UNION 模拟

```sql
-- 全外连接 = 左连接 UNION 右连接（UNION 去重，列数与类型必须一致）
SELECT u.name, o.order_no
FROM t_user u
LEFT JOIN t_order o ON o.user_id = u.id
UNION
SELECT u.name, o.order_no
FROM t_user u
RIGHT JOIN t_order o ON o.user_id = u.id;
```

### 左连接最容易错的一点：条件放错位置

```sql
-- 错：右表条件写进 WHERE，左连接退化成内连接（没有订单的用户被滤掉了）
SELECT u.name, o.order_no
FROM t_user u
LEFT JOIN t_order o ON o.user_id = u.id
WHERE o.status = 1;

-- 对：右表条件写进 ON，过滤的是“匹配规则”不是“结果行”
SELECT u.name, o.order_no
FROM t_user u
LEFT JOIN t_order o ON o.user_id = u.id AND o.status = 1;
```

- 口诀：ON 定“怎么连”，WHERE 定“连完留哪些行”。左连接下，右表条件放 WHERE 会把右表为 NULL 的行（即左表无匹配的行）一起滤掉。
- 多表连接时先想驱动关系：从“必须全出”的表出发画 LEFT 链；能用内连接就不用左连接（内连接给优化器的选择更多）。

### 半连接与反连接：IN / EXISTS 的选择

```sql
-- 半连接：只要“存在订单”的用户（不关心订单本身）
SELECT * FROM t_user u WHERE EXISTS (
  SELECT 1 FROM t_order o WHERE o.user_id = u.id
);

-- 反连接：只要“从没下过单”的用户
SELECT * FROM t_user u WHERE NOT EXISTS (
  SELECT 1 FROM t_order o WHERE o.user_id = u.id
);
```

- `IN (子查询)` 与 `EXISTS` 语义接近时优先 `EXISTS`：子查询表大时 `EXISTS` 找到一行就停；`IN` 要注意子查询结果含 NULL 时整个 `IN` 判定会变 UNKNOWN（见第 8 节）。
- 只查“有没有”，不要 `JOIN` 后再 `DISTINCT` 去重：JOIN 会先把行数乘出来再压回去，又慢又容易错。

## 4. 聚合与 GROUP BY / HAVING

```sql
-- 分组统计：每种状态的订单数与总金额
SELECT status, COUNT(*) AS cnt, SUM(amount) AS total
FROM t_order
GROUP BY status
HAVING COUNT(*) > 10        -- 过滤“分组结果”，不是过滤行
ORDER BY total DESC;

-- WITH ROLLUP：分组结果后追加一行总计（status 为 NULL 的那行就是总计）
SELECT status, COUNT(*) AS cnt, SUM(amount) AS total
FROM t_order
GROUP BY status WITH ROLLUP;
```

- **WHERE 与 HAVING 的分工**：WHERE 在分组前过滤行（能写 WHERE 的绝不写 HAVING）；HAVING 在分组后过滤分组（只能用分组列和聚合函数）。
- **ONLY_FULL_GROUP_BY**：MySQL 5.7.5 起默认开启，8.0 延续。GROUP BY 后 SELECT 只能出分组列和聚合结果——想出非分组列要么加进 GROUP BY，要么用聚合函数包起来（如 `MAX(name)`），要么用 `ANY_VALUE()` 显式声明“随便取一行”。
- **COUNT 的三种写法**：`COUNT(*)` 计行数（含 NULL 行）；`COUNT(col)` 只计该列非 NULL 的行；`COUNT(1)` 与 `COUNT(*)` 等价。想统计“有多少用户下过单”用 `COUNT(DISTINCT user_id)`，别用 `COUNT(*)`。
- **聚合 + 明细混写是错的**：`SELECT user_id, SUM(amount) FROM t_order` 在 ONLY_FULL_GROUP_BY 下直接报错；既要明细又要汇总，用窗口函数（第 6 节）或两条查询。

## 5. 子查询与 CTE

### 子查询的三种位置

```sql
-- 标量子查询：出现在 SELECT 里，一次返回一个值
SELECT id, order_no,
  (SELECT name FROM t_user WHERE id = o.user_id) AS user_name
FROM t_order o;

-- 表子查询：出现在 FROM 里，当成一张临时表（必须起别名）
SELECT t.city, AVG(t.avg_amount) AS city_avg
FROM (
  SELECT u.city, o.user_id, AVG(o.amount) AS avg_amount
  FROM t_order o JOIN t_user u ON o.user_id = u.id
  GROUP BY u.city, o.user_id
) t
GROUP BY t.city;

-- 条件子查询：出现在 WHERE 里（见第 3 节 EXISTS）
SELECT * FROM t_order WHERE user_id IN (SELECT id FROM t_user WHERE city = '深圳');
```

- 子查询第一反应应该是“能不能改成 JOIN”：JOIN 通常更易读、优化器选择更多；子查询留给“确实需要先算出一个集合再过滤”的场景。
- `IN (子查询)` 的 NULL 坑：子查询结果含 NULL 时，`x IN (...)` 对不匹配的 x 返回 UNKNOWN 而非 FALSE，`NOT IN` 会直接查不出任何行——反连接优先用 `NOT EXISTS`。

### CTE：给子查询起名字

```sql
-- 普通 CTE：多段逻辑拆成可读的步骤，MySQL 8.0+
WITH user_total AS (
  SELECT user_id, SUM(amount) AS total FROM t_order GROUP BY user_id
)
SELECT u.name, t.total
FROM user_total t
JOIN t_user u ON u.id = t.user_id
WHERE t.total > 1000;

-- 递归 CTE：查层级/链条，如组织架构、分类树，MySQL 8.0+
WITH RECURSIVE org AS (
  SELECT id, name, manager_id FROM t_employee WHERE id = 1   -- 锚点：起点行
  UNION ALL
  SELECT e.id, e.name, e.manager_id
  FROM t_employee e JOIN org o ON e.manager_id = o.id        -- 递归步：沿关系向下
)
SELECT * FROM org;
```

- CTE vs 派生表：CTE 可读性更好、可复用（同一 CTE 在一条语句里引用多次）、支持递归；派生表适合只用一次的简单场景。
- 递归 CTE 必须有终止条件（锚点 + 递进条件），否则会一直递归到 `cte_max_recursion_depth` 报错；`UNION ALL` 保留重复行，`UNION` 会去重（层级查询一般用 `UNION ALL`）。

## 6. 窗口函数

窗口函数（MySQL 8.0+）是“分组计算但不压行”：聚合函数会把一组压成一行，窗口函数算完后每一行都还在，算出的值挂在行上。

### 语法骨架

```sql
<窗口函数> OVER (
  [PARTITION BY 列]              -- 分组：不写 = 全表一个组
  [ORDER BY 列]                  -- 组内排序：排名类函数必须写
  [ROWS | RANGE BETWEEN ...]    -- 窗口帧：算到哪几行，默认到当前行
)
```

### 排名函数：三者的区别是核心考点

```sql
SELECT user_id, amount,
  ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn,   -- 1,2,3,4：绝不并列
  RANK()       OVER (ORDER BY amount DESC) AS rk,   -- 1,2,2,4：并列占位，后面跳号
  DENSE_RANK() OVER (ORDER BY amount DESC) AS dr    -- 1,2,2,3：并列不占位，后面不跳号
FROM t_order;
```

- 分组取 Top N 的标准写法（先排名、再外层过滤，窗口函数不能直接写进 WHERE）：

```sql
-- 每个用户金额最大的 3 笔订单
SELECT * FROM (
  SELECT o.*,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
  FROM t_order o
) t
WHERE t.rn <= 3;
```

### 聚合窗口：明细与汇总同屏

```sql
-- 每行挂上“该用户累计金额”和“该用户订单总额”，行数不变
SELECT id, user_id, amount,
  SUM(amount) OVER (PARTITION BY user_id) AS user_total,
  SUM(amount) OVER (
    PARTITION BY user_id ORDER BY create_time
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM t_order;
```

- **ROWS 与 RANGE 的区别**：ROWS 按物理行数切帧（前 N 行）；RANGE 按 ORDER BY 的值切帧（值相同的行同进同出）。`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 在有并列值时会把并列行一起算进累计，ROWS 则不会——累计值要求“逐行递增”时用 ROWS。
- 不写帧子句时默认帧是 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`（到当前行，含并列值），不是“整组”。

### 偏移与分布函数

```sql
SELECT id, user_id, amount, create_time,
  LAG(amount)  OVER (PARTITION BY user_id ORDER BY create_time) AS prev_amount,
  LEAD(amount) OVER (PARTITION BY user_id ORDER BY create_time) AS next_amount,
  amount - LAG(amount) OVER (PARTITION BY user_id ORDER BY create_time) AS diff_prev,
  NTILE(4)     OVER (ORDER BY amount DESC) AS quartile,   -- 均分 4 档：1 最好
  PERCENT_RANK() OVER (ORDER BY amount DESC) AS pct_rank, -- 相对位置 0~1
  CUME_DIST()    OVER (ORDER BY amount DESC) AS cum_dist  -- 小于等于当前值的比例
FROM t_order;
```

- LAG/LEAD 的典型场景：环比/同比（上一行 vs 本行）、相邻行差值、状态变化检测（本行状态 ≠ 上一行状态即变化点）。
- LAG/LEAD 的第二个参数是偏移量（默认 1）、第三个参数是越界默认值：`LAG(amount, 1, 0)` 表示第一行没有上一行时取 0。
- NTILE 分档不均匀时前面的档多分一行；PERCENT_RANK 的 0 不代表“最差”，只代表“排第一”。

### 窗口函数的执行位置

- 窗口函数在 WHERE、GROUP BY 之后执行，所以不能直接写进 WHERE 或 GROUP BY——要过滤窗口结果必须包一层子查询或 CTE（见分组 Top N 的写法）。
- 一个 SELECT 里可以有多个窗口函数、不同 PARTITION，互不干扰。

## 7. SQL 执行顺序

书写顺序：SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT。

逻辑执行顺序（决定语义的顺序）：

```
FROM / JOIN → ON → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

由此推出的三条铁律：

1. **WHERE 里不能用 SELECT 别名**（别名在 SELECT 阶段才产生，WHERE 在它之前）；ORDER BY 可以用别名（ORDER BY 在 SELECT 之后）。MySQL 对 GROUP BY / HAVING 用别名做了宽松处理，但别依赖它，跨库不通用。
2. **WHERE 里不能用聚合函数和窗口函数**（它们分别在 GROUP BY / SELECT 之后才产生）；聚合条件走 HAVING，窗口结果走外层过滤。
3. **LIMIT 在最后执行**：`ORDER BY ... LIMIT 20` 是“先排好序再取前 20”，不是“取 20 再排”；分页越往后越慢是性能问题（见调优篇），不是语义问题。

## 8. 常见误区

1. **`= NULL` 永远查不到**：NULL 不是值，是“未知”，判空用 `IS NULL` / `IS NOT NULL`；`NULL = NULL` 也是 UNKNOWN。
2. **WHERE 里写聚合或窗口函数**：见第 7 节铁律 2，聚合条件用 HAVING，窗口结果包一层再过滤。
3. **左连接条件放错位置**：右表条件放 WHERE 会退化成内连接（第 3 节），放 ON 才保住左表全量。
4. **GROUP BY 后 SELECT 非分组列**：ONLY_FULL_GROUP_BY 下报错；要么进 GROUP BY，要么包聚合函数，要么改用窗口函数。
5. **NOT IN 遇 NULL 全灭**：子查询含 NULL 时 `NOT IN` 返回空集；反连接用 `NOT EXISTS`。
6. **`BETWEEN` 是闭区间**：`BETWEEN 100 AND 500` 含 100 和 500；日期区间用 `>= 开始 AND < 结束` 更稳（`BETWEEN '2026-01-01' AND '2026-01-31'` 会漏掉 31 号当天的时分秒部分）。
7. **字符串列用数字查**：`WHERE order_no = 2026001`（order_no 是 VARCHAR）会触发隐式转换，索引可能失效且结果可能错位——类型对齐是写对的一部分，完整讲法见调优篇。
8. **LIKE 前导 `%` 走不了索引**：`LIKE '张%'` 可走索引，`LIKE '%张'` 不行；要全文检索需求用全文索引或 ES，别硬拼 LIKE。
9. **UPDATE / DELETE 忘 WHERE**：见第 2 节，先 SELECT 确认行数再执行；生产删改优先逻辑删除。
10. **把“能跑”当成“写对”**：小数据量下错的 SQL 也能跑出看似正确的结果（排序不稳定、NULL 判定、帧语义），上线前用边界数据（NULL、并列值、空组、单行组）各测一遍。

## 面试与自测

1. 左连接里把右表条件写进 WHERE 和写进 ON，结果有什么不同？为什么？
2. ROW_NUMBER、RANK、DENSE_RANK 在并列值上的行为各是什么？分组取 Top N 为什么要包一层子查询？
3. WHERE、HAVING、窗口函数结果过滤三者的执行位置关系是什么？`SELECT user_id, SUM(amount) FROM t_order WHERE SUM(amount) > 1000` 为什么报错？

### 参考答案

1. 不同。条件写进 ON 时，过滤的是“匹配规则”，左表无匹配的行仍保留（右表字段为 NULL）；条件写进 WHERE 时，过滤的是“连接后的结果行”，右表为 NULL 的行会被 `WHERE 右表条件` 滤掉，左连接退化成内连接。
2. ROW_NUMBER 绝不并列（1,2,3,4）；RANK 并列占位、后面跳号（1,2,2,4）；DENSE_RANK 并列不占位、后面不跳号（1,2,2,3）。分组取 Top N 要包一层子查询，是因为窗口函数在 WHERE 之后执行，不能直接写进 WHERE，只能先算出排名再在外层过滤。
3. 执行顺序是 WHERE（分组前过滤行）→ GROUP BY → HAVING（分组后过滤分组）→ SELECT（含窗口函数计算）→ ORDER BY → LIMIT。`WHERE SUM(amount) > 1000` 报错是因为 WHERE 执行时聚合还没算出来，聚合条件必须写进 HAVING；同理窗口函数的结果也不能直接进 WHERE，要包一层子查询或 CTE 再过滤。

## 关联知识

- [MySQL 数据库](数据库/MySQL/MySQL-数据库.md)：DML 速记、JOIN 三连、聚合分组速记、事务四句、类型速选。
- [MySQL 索引与执行计划](数据库/MySQL/MySQL-索引与执行计划.md)：写对之后怎么变快，EXPLAIN 完整读法。
- [SQL调优实战](数据库/MySQL/SQL调优实战.md)：慢 SQL 闭环、隐式转换、深分页等性能侧讲法。
