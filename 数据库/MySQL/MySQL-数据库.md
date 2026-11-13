---
知识点: MySQL 数据库
层级: 进阶
适用版本: MySQL 5.7 / 8.x
更新日期: 2026-11-14
关联:
  - 数据库知识点地图
  - MySQL-binlog
  - Redis
  - Elasticsearch
标签:
  - 数据库
  - 索引
  - 原理
  - 实践
---

# MySQL 数据库

> 每晚 20:30 自动补充完善。项目用 OceanBase MySQL 模式，兼容 MySQL 5.7 语法，绝大多数经验直接通用；OB 特有差异见文末。

## 索引速记（深挖见专题篇）

- **最左前缀**：联合索引 `(a, b, c)` 必须从最左列开始连续使用，缺 a 则 b、c 都用不上。
- **失效速记**：索引列做函数/运算、`varchar` 列用数字查（隐式转换）、`LIKE '%xxx'`、OR 一边没索引、`!=`/`NOT IN`、优化器判断全表更快。
- **覆盖索引**：查询列全在索引里就不回表；索引下推（ICP）能在索引层先过滤再回表。
- **EXPLAIN 速看**：type 由好到差 `const` > `ref` > `range` > `index` > `ALL`，见 `ALL` 先警觉；key 为 NULL 是没用上；Extra 的 `Using index` 是好事，`Using filesort` / `Using temporary` 尽量消掉。

原理、索引设计口诀、EXPLAIN 完整读法与慢 SQL 闭环见专题篇：[MySQL 索引与执行计划](数据库/MySQL/MySQL-索引与执行计划.md)。

## 慢查询排查速记

1. 先开慢查询日志圈定 Top SQL，再用 `EXPLAIN` 看计划——完整流程见上行专题篇。
2. **深分页**（总篇独有要点）：`LIMIT 1000000,20` 越翻越慢，改游标分页（`WHERE id>上次id LIMIT 20`）。
3. 索引没问题还慢，再查大事务、锁等待（`SHOW ENGINE INNODB STATUS`）和连接数；事务与锁深挖见专题篇：[MySQL 事务与 MVCC](数据库/MySQL/MySQL-事务与MVCC.md)。

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

## 疑难杂症

### 1. OceanBase 与 MySQL 的 SQL 兼容差异
- **现象**：同一套 SQL 在 MySQL 跑得好好的，OceanBase 报错或结果不对。
- **原因**：OceanBase MySQL 模式兼容约 90%+，但窗口函数、部分 DDL、系统变量、事务行为有差异。
- **解决**：以 OceanBase 官方兼容性文档为准；联调阶段用 `EXPLAIN` 对比执行计划；新项目优先用两者都支持的标准写法。

### 2. 死锁排查
- **现象**：并发一高就报 `Deadlock found when trying to get lock`，日志里看不出谁和谁打架。
- **原因**：两个事务以不同顺序加锁（如 A 先锁行 1 再锁行 2，B 反过来）。
- **解决**：`SHOW ENGINE INNODB STATUS` 看 `LATEST DETECTED DEADLOCK` 段，找加锁顺序；业务层统一加锁顺序；缩短事务，能不放事务里的查询拿出来。

### 3. 主从延迟导致读到旧数据
- **现象**：刚写入的数据，读接口查出来是旧的，刷新几次才正常。
- **原因**：写主库、读从库，从库复制有延迟。
- **解决**：写后立即读走主库（`forceMaster`）；或业务接受最终一致；延迟持续偏高查从库的 SQL 线程/IO 线程状态。

## MySQL 8.4 LTS（2026-10-18 核验）

- **8.0 在 2026-04 EOL**，新项目直接上 8.4 LTS（支持到 2032），别再装 8.0。
- 默认认证改 `caching_sha2_password`（`mysql_native_password` 默认禁用），老客户端连不上先查这个。
- InnoDB 默认值按现代 SSD 重调：`innodb_io_capacity` 200→10000，`innodb_log_buffer_size` 16MB→64MB，`innodb_flush_method` 默认 O_DIRECT。
- 删掉了 MASTER/SLAVE 术语（`SHOW MASTER STATUS` 等改名），旧运维脚本要批量改。
- OceanBase 用户注意：OceanBase MySQL 模式对标的是 5.7/8.0 语法子集，8.4 的新函数/新变量不一定兼容，联调时以 OceanBase 官方兼容性矩阵为准。
- 来源：https://dev.mysql.com/doc/refman/8.4/en/mysql-releases.html

## OceanBase 租户与资源单元（导读）

链条是集群 > 租户 > 数据库，不等于 MySQL 的"实例 > 库"：租户是资源隔离单位（配额独立），资源单元（UNIT）是最小粒度，扩容 = 加 UNIT。神农连的是 MySQL 模式租户，`SHOW VARIABLES` 是租户级变量（如 `ob_trx_timeout` 只影响当前租户）；排障先看 `DBA_OB_UNITS` 配额与租户水位。完整展开见专题篇：[OceanBase](数据库/OceanBase.md)。

## 分库分表中间件选型

| 方案 | 类型 | 特点 |
|---|---|---|
| ShardingSphere-JDBC | 客户端 jar | 无额外运维，应用直连，适合中小规模 |
| ShardingSphere-Proxy | 代理 | 对应用透明（当 MySQL 用），可独立扩缩 |
| MyCat | 代理 | 老牌，社区活跃度下降，新项目不推荐 |

- 分片键选查询最常用的维度（用户 ID），别选时间（热点都挤在最新分片）。
- 能不分就不分：先读写分离 + 缓存，分表是最后手段（跨分片 join/事务是深渊）。
- 分了之后主键用雪花算法，别用自增（各分片自增会冲突）。

## binlog（已拆分）

binlog 的格式与参数、事件链、mysqlbinlog 解码、主从复制、CDC、PITR 恢复与闪回、Java 订阅示例已拆为独立知识点：[MySQL binlog](数据库/MySQL/MySQL-binlog.md)。

## 执行计划进阶：join 算法

```sql
EXPLAIN FORMAT=TREE SELECT * FROM a JOIN b ON a.id=b.a_id;
```

- **Nested Loop**：小表驱动大表，`b.a_id` 必须有索引，不然每次全表扫。
- **Hash Join**（8.0.18+）：无索引等值 join 的救星，内存够就快，`join_buffer_size` 给大点。
- **BNL**（Block Nested Loop）：没索引时的保底，慢，看到它就去加索引。
- `EXPLAIN` 看到 `Using filesort` + 大数据量：加复合索引让"排序"变"顺序读"。

## MySQL 9.7 LTS / 8.4 时效更新（2026-10-21 核验）

### 已验证事实

1. **当前最新补丁（2026-10-21 实测）**：9.7.2 LTS、8.4.13 LTS。Innovation 线已改日历版本号：26.7.0（2026-07 GA），10 月季度发布为 26.10.0。
   - 来源：https://sqlmirror.com/（2026-10-21 实测）、https://blogs.oracle.com/mysql/mysql-july-2026-ga-releases-now-available
2. **8.0 已 EOL（2026-04）**：Ubuntu 24.04 官方源里还躺着 8.0.46，`apt install mysql-server` 会装到 EOL 版本——生产环境一律走 Oracle APT/YUM 源，装 8.4 或 9.7 LTS。
   - 来源：https://www.bytebase.com/blog/postgres-vs-mysql/（2026-09-23 修订）、https://computingforgeeks.com/install-mysql-ubuntu-debian/（2026-10 实测）
3. **9.7 LTS 的 headline：Hypergraph Optimizer 进了社区版**。多表深 join / 星型模型 / 分析型查询可做数量级优化；按语句开启最稳妥，全局开是错的（简单 OLTP 查询规划成本反而上升）。
   ```sql
   SET optimizer_switch = 'hypergraph_optimizer=on';          -- 会话级
   SELECT /*+ SET_VAR(optimizer_switch = 'hypergraph_optimizer=on') */ ...;  -- 语句级
   ```
   - 来源：https://www.bytebase.com/blog/notes-on-mysql-9-7-lts-release/（2026-05）
4. **9.x 彻底删了 `mysql_native_password`**（8.4 只是默认禁用、可 opt-in）。Java 老驱动/老连接池连 9.x 报错先查认证插件和驱动版本，`mysql-connector-j` 先升到 8.4+。
   - 来源：https://computingforgeeks.com/install-mysql-ubuntu-debian/（2026-10 实测）
5. **升级路径硬约束**：8.4.x LTS → 9.7.x LTS → 26.x 日历线。8.4 不能直接跳到 26.x。下一个 LTS 是 28.4.0，预计 2028-04。
   - 来源：https://www.percona.com/blog/meet-mysql-26-7-a-new-versioning-era-begins/、https://blogs.oracle.com/mysql/a-more-predictable-mysql-release-model-calendar-versions-lts-and-innovation
6. **9.7 LTS 其他值得知道的**：JSON Duality Views（关系+JSON 混合视图）、Group Replication 可观测性增强（Performance Schema 新表）、OpenTelemetry 监控。
   - 来源：https://www.bytebase.com/blog/notes-on-mysql-9-7-lts-release/
7. **OceanBase 4.x 近况**：V4.6.0 的 Release Notes 与 AI（向量）文档已发布；MySQL 模式兼容对标的是 MySQL 8.0 语法子集（4.1 文档确认），窗口函数、系统变量、事务行为仍有差异。
   - 来源：OceanBase 官方文档 PDF（V4.6.0，2026-09 发布）：https://obportal.s3.ap-southeast-1.amazonaws.com/doc/product_pdf/oceanbase-database/276/OceanBase-Database-V4.6.0-Release-Notes.pdf

### 个人推断（供参考，别当事实引用）

- 神农项目（OceanBase MySQL 模式）短期内跟进 MySQL 8.4 新函数/新变量意义不大：OB 对标的是 5.7/8.0 子集，8.4/9.7 的新语法在 OB 上大概率不生效，SQL 写法继续按 8.0 及更早的公共子集写最保险。
- 新项目如果不用 OB、直接上社区 MySQL：选 9.7 LTS（支持周期比 8.4 晚两年启动）；已有 8.4 集群：8.4.13 补丁跟上即可，没必要追 9.7。

### 待验证

- 10 月季度 CPU 后 9.7 的补丁号是否为 9.7.3、8.4 是否为 8.4.14（2026-10-21 当天实测仍是 9.7.2 / 8.4.13，下一轮夜任务再核一次）。
- OceanBase V4.6.0 的确切 GA 日期与 MySQL 8.0 兼容性矩阵是否有变化（以 https://www.oceanbase.com/docs 为准）。

## 基础知识补强

### 核心概念与术语

- 心智模型：实例 > 数据库（schema）> 表 > 行 > 列；一行是一条记录，一列是一个字段，表结构先于数据存在。
- 主键：唯一标识一行，不能为 NULL，一张表只能有一个（可多列联合）；外键：引用另一张表主键的列，用来保证两张表的数据对得上。
- 约束：建表时给列定的规矩——主键、唯一（UNIQUE）、非空（NOT NULL）、默认值（DEFAULT）、外键（FOREIGN KEY），入库前先校验，不合规直接报错。
- 事务四特性 ACID：原子性（全成或全不成）、一致性（数据不自相矛盾）、隔离性（并发事务互不干扰到约定程度）、持久性（提交了就不丢）。
- 范式：拆表的基本准则，记结论即可——1NF：每列不可再分；2NF：非主键列完全依赖整个主键；3NF：非主键列不依赖另一个非主键列（消除「订单表里冗余存用户名」这类传递依赖）。实际业务常适度反范式换查询性能，但要先懂范式再谈反范式。
- 字符集与排序规则：字符集决定能存哪些字符（统一用 utf8mb4）；排序规则（collation，如 utf8mb4_general_ci）决定字符串怎么比大小，ci = 大小写不敏感，两表 JOIN 时排序规则不一致会报错或索引失效。

常用数据类型速选：

| 类别 | 类型 | 用法 |
|---|---|---|
| 整数 | TINYINT / INT / BIGINT | 状态用 TINYINT、ID 用 BIGINT，别用 INT 存手机号（会溢出且丢前导 0） |
| 小数 | DECIMAL(M,D) | 存钱必须 DECIMAL；FLOAT/DOUBLE 有精度误差，只存测量值 |
| 字符串 | VARCHAR(n) | 变长短文本；n 按实际设计，别无脑 255 |
| 文本 | TEXT | 长文本，不设默认值、少建索引，和主表查询分开考虑 |
| 日期 | DATETIME / TIMESTAMP | DATETIME 存绝对时间；TIMESTAMP 带时区转换、范围小，跨时区业务慎用 |
| 枚举替代 | TINYINT + 字典 | 别用 ENUM，加值要改表 |

### 基础语法与用法

```sql
-- 库与表的基础操作
CREATE DATABASE shop DEFAULT CHARACTER SET utf8mb4;
USE shop;
SHOW TABLES;
DESC t_order;                 -- 看列定义
SHOW CREATE TABLE t_order;    -- 看完整建表语句，排查字符集/引擎最准

-- 建表示例：约束写全
CREATE TABLE t_order (
  id          BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键',
  user_id     BIGINT       NOT NULL COMMENT '下单用户',
  order_no    VARCHAR(32)  NOT NULL COMMENT '订单号',
  amount      DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '金额',
  status      TINYINT      NOT NULL DEFAULT 0 COMMENT '0待支付 1已支付',
  create_time DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表';

-- DML 四件套
INSERT INTO t_order (user_id, order_no, amount) VALUES (1001, 'NO2026001', 99.00);
SELECT id, order_no, amount FROM t_order WHERE user_id = 1001 AND status = 0;
UPDATE t_order SET status = 1 WHERE id = 10;          -- 必须带 WHERE
DELETE FROM t_order WHERE id = 10;                    -- 必须带 WHERE（见易错点）

-- 排序、分页、去重
SELECT * FROM t_order WHERE user_id = 1001 ORDER BY create_time DESC LIMIT 20;
SELECT DISTINCT status FROM t_order;

-- 聚合与分组（GROUP BY 后 SELECT 只能出分组列和聚合结果）
SELECT status, COUNT(*) AS cnt, SUM(amount) AS total
FROM t_order GROUP BY status HAVING COUNT(*) > 10;    -- HAVING 过滤分组结果，WHERE 过滤行

-- JOIN 三连：内连接（两边都有才出）、左连接（左表全出）、右连接（少用，换成左连）
SELECT o.order_no, u.name
FROM t_order o
LEFT JOIN t_user u ON o.user_id = u.id;

-- 子查询：先想清楚能不能改成 JOIN，子查询可读性差且常更慢
SELECT * FROM t_user WHERE id IN (SELECT user_id FROM t_order WHERE amount > 1000);

-- 事务四句：多条语句要么一起成、要么一起退
START TRANSACTION;
UPDATE t_account SET balance = balance - 100 WHERE id = 1;
UPDATE t_account SET balance = balance + 100 WHERE id = 2;
COMMIT;                                  -- 确认无误才提交；出问题用 ROLLBACK 回滚
-- SAVEPOINT s1 / ROLLBACK TO s1：事务内打点，只回滚后半段

-- 基础条件速查：IN（多值任一）、BETWEEN（闭区间）、LIKE（%任意字符 _单个字符）
SELECT * FROM t_user WHERE city IN ('深圳', '广州');
SELECT * FROM t_order WHERE amount BETWEEN 100 AND 500;
SELECT * FROM t_user WHERE name LIKE '张%';           -- 前缀匹配可走索引，'%张' 不行

-- 改表结构 ALTER（大表慎用，上线前先在测试环境试；加列带默认值在新版多为秒级，老版本可能锁表）
ALTER TABLE t_order ADD COLUMN remark VARCHAR(128) NOT NULL DEFAULT '';
ALTER TABLE t_order ADD INDEX idx_status_time (status, create_time);
ALTER TABLE t_order MODIFY COLUMN remark VARCHAR(256) NOT NULL DEFAULT '';

-- 授权基础（DCL）
CREATE USER 'app'@'%' IDENTIFIED BY '******';
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app'@'%';  -- 按最小权限给，别给 *.*
```

### 关键机制速览

- 存储引擎：默认 InnoDB——支持事务、行锁、外键；MyISAM 无事务、只有表锁，只在只读老系统里见到，新表一律 InnoDB。
- 一条查询的旅程：连接器鉴权 → 查询缓存（8.0 已移除）→ 解析/优化器选执行计划 → 执行器按计划调存储引擎取行 → 返回。慢查询多数卡在优化器选错索引或引擎扫太多行。
- 连接与会话：每个连接是一个会话，`SET` 的变量分会话级和全局级；连接池的意义是复用连接，避免频繁三次握手 + 鉴权。
- 自增 ID 的本质：表级计数器，保证本表单调递增，不保证连续（回滚、删除都会留空洞），更不能当业务序号用。

### 新手易错点

- UPDATE / DELETE 忘写 WHERE：全表被改/删。养成习惯：先用相同 WHERE 跑一遍 SELECT 确认行数，再执行；生产删改优先逻辑删除（加 del_flag）。
- DROP TABLE / TRUNCATE vs DELETE：DROP 连表结构一起删，TRUNCATE 清空全表且不可回滚、自增归零——危险命令，执行前必须确认连的是哪个库、表里数据要不要留档；DELETE 是逐行删、可回滚、可带 WHERE。
- 用 `= NULL` 判断空：永远查不到，判空用 `IS NULL` / `IS NOT NULL`。
- 把聚合条件写进 WHERE：WHERE 在分组前执行，不能写 COUNT() 这类聚合，过滤分组结果用 HAVING。
- SELECT *：取回无用列、覆盖索引失效、表结构变更时代码易炸，按需列出字段。
- 钱用 FLOAT 存、时间用字符串存、枚举用中文字符串存：三类经典建模错误，分别改 DECIMAL、DATETIME、TINYINT + 字典表。
- VARCHAR 长度按字符算不是字节：utf8mb4 下一个汉字占 3~4 字节但算 1 个字符，`VARCHAR(255)` 指 255 个字符，别按字节估算。

## 原理剖析

本节只把原文各处已经给出的结论串起来，讲清它们背后的机制，不引入原文没有的新数字、新 API 与新版本行为。凡原文已标明来源或口径的，沿用原文的标注。

### 索引：B+ 树查找、回表与覆盖索引

- **查找是“按键定位，再按需取行”**：按索引键在 B+ 树里逐层定位到叶子，找到符合条件的索引项；索引项里只存索引列与主键，查询要的列如果不在索引里，就拿主键再回表取一整行，这一步就是回表。回表不是错误，是“索引里没有要的列”时的必然补查；回表次数一多，索引带来的收益就被抵消了。
- **覆盖索引省掉的就是回表**：原文索引速记的结论是“查询列全在索引里就不回表”。按需列出字段而不是 `SELECT *`，原因也在这里——多取一列无关列，就可能让一次本可覆盖的查询变成回表查询。`EXPLAIN` 的 Extra 出现 `Using index`，就是优化器认为这次查询可以只读索引、不回表。
- **最左前缀是“连续可用”的意思**：联合索引 `(a, b, c)` 必须从最左列开始连续使用，缺 a 则 b、c 都用不上。跳过左列去用右列，定位时没有连续的键范围可走，优化器只能放弃这部分索引；这与“索引列做函数/运算、隐式转换、`LIKE '%xxx'` 导致失效”是同一类原因——条件写出来之后，优化器无法再按索引键的顺序去定位。
- **索引下推（ICP）是“先过滤、再回表”**：能在索引层先过滤掉不符合条件的索引项，只对剩下的项回表。回表次数少了，但每行仍要回表取数；它省的是回表的量，不是回表本身。
- **失效的另一面是“优化器觉得全表更快”**：原文失效速记里有一条“优化器判断全表更快”。索引不是越用越好：要回的行占表里比例太高时，逐行回表的代价超过顺序扫全表，优化器就会放弃索引。这时 `EXPLAIN` 的 type 掉到 `ALL`、key 为 NULL，先警觉，再看是条件写错了，还是这条查询本来就不该走索引。

### 事务与锁：MVCC 可见性、行锁与死锁

- **快照读与加锁读是两回事**：普通 `SELECT` 读的是事务开始时的快照，不加锁；`SELECT ... FOR UPDATE` 是对读到的行加排他锁，`LOCK IN SHARE MODE`（8.0 里叫 `FOR SHARE`）是加共享锁。原文小技巧的结论是“读完马上要改”的逻辑必须用 `FOR UPDATE`——只用快照读，两个事务会各自读到同一份旧值、各自算出一份扣减，并发下就多扣了。
- **可见性的代价落在 undo 上**：快照要靠旧版本数据留着才能读到，事务迟迟不提交，旧版本就迟迟不能回收。原文常见坑的结论是“长事务拖死 undo / 锁”——长事务的危害不只是它自己占着锁，更是它拖住了一整段旧版本，让别的事务的清理也走不动。事务里别调远程接口、别 sleep，缩短事务就是缩短这段拖住的时间。
- **行锁锁的是行，不是表**：InnoDB 默认行锁，两个事务改不同行互不干扰；改同一行时后到的等先到的提交或回滚。`SHOW ENGINE INNODB STATUS` 能看到锁等待，慢查询排查里“索引没问题还慢，再查大事务、锁等待”就是这个顺序：先排除计划问题，再看是不是有人在等锁。
- **死锁是“加锁顺序成环”**：原文疑难杂症的结论是两个事务以不同顺序加锁（如 A 先锁行 1 再锁行 2，B 反过来），互相等对方手里的锁，谁也走不下去。排查看 `LATEST DETECTED DEADLOCK` 段找加锁顺序，根治靠业务层统一加锁顺序、缩短事务、把能不放事务里的查询拿出来。死锁不是锁不够用，是顺序撞上了；加更多锁、调大超时都解不了顺序问题。
- **间隙锁（原文未展开，仅作概念提示）**：原文只讲到行锁与死锁成因，没有单独展开间隙锁的加锁规则与触发条件，本节不展开。只记一条边界：涉及范围条件的加锁读，其加锁范围可能超出命中的行本身，排查锁等待时除了看“谁锁了哪行”，也要看语句里的范围条件；具体规则以官方手册与专题篇为准，不要按本节的只言片语推断。
- **隔离性是“约定程度”，不是绝对互不干扰**：原文 ACID 里隔离性的定义是“并发事务互不干扰到约定程度”。快照读、加锁读、长事务拖 undo、死锁，都是这个“约定程度”在实现层面的不同侧面；OceanBase 下还要再打一个折扣——单分区内事务强一致，跨分区事务是弱隔离，写代码时别假设单机 MySQL 的锁行为。

### 执行计划：join 算法是怎么选的

- **Nested Loop：小表驱动大表，被驱动表必须有索引**：原文执行计划进阶的结论是 `b.a_id` 必须有索引，不然每次全表扫。机制是外层每取一行，就到内层按连接键查一次；内层没有索引，一次就是一次全表扫，外层有多少行就扫多少次。驱动表选小表，是为了让“查内层的次数”尽可能少。
- **Hash Join（8.0.18+）：无索引等值 join 的救星**：没有可用索引时，把一张表按连接键建成哈希表，另一张表逐行探查，省掉反复扫表。原文的限定是“内存够就快，`join_buffer_size` 给大点”——哈希表要放进内存才快，内存不够时优势会打折扣；它只解决等值 join，没有索引也不是无条件地快。
- **BNL（Block Nested Loop）：没索引时的保底，慢**：原文的结论是“看到它就去加索引”。它是 Nested Loop 在没索引时的退化形态：内层没有索引可查，只能按块反复扫，代价随两表行数相乘式地涨。执行计划里出现 BNL，问题不在 join 算法本身，在缺失的索引。
- **选错的信号在 `EXPLAIN` 里**：type 由好到差 `const` > `ref` > `range` > `index` > `ALL`，见 `ALL` 先警觉；key 为 NULL 是没用上；Extra 的 `Using filesort` / `Using temporary` 尽量消掉。`EXPLAIN` 只给预估，怀疑优化器选错索引时上 `EXPLAIN ANALYZE`（8.0.18+）看实际执行——它真的跑一遍查询，打出每一步的实际耗时和实际行数，优化器“以为很快、实际很慢”的误判一眼就能看出来；别一上来就对生产库跑它。
- **排序也可以“变顺序读”**：`EXPLAIN` 看到 `Using filesort` + 大数据量，加复合索引让“排序”变“顺序读”。机制与覆盖索引是同一条：索引本身是有序的，按索引顺序取行就不需要再排一次；复合索引的列顺序要与排序键对上，否则优化器还是得另起一次排序。

## 连接与线程定量：连接数、线程缓存与每连接内存算一算

### 核心概念与默认值

- **一个连接就是一个会话、一个线程**：经典模型是一连接一线程（`thread_handling = one-thread-per-connection`），连接不断开，线程和会话内存就不释放。先分清三个数：`Threads_connected`（当前打开的连接数，含空闲）、`Threads_running`（正在执行 SQL 的连接数）、`max_connections`（允许同时存在的连接数上限）。
- **`max_connections` 默认 151**：MySQL 8.0 与 8.4 官方手册均核实为 151，最大值 100000。打满后再连会报 `ERROR 1040 (08004): Too many connections`。另有 1 个预留的管理连接：持有 `CONNECTION_ADMIN`（或已废弃的 `SUPER`）权限的账号在打满后仍能连上排障，别把这个救命通道随便授给业务账号。
- **`thread_cache_size` 是线程缓存，不是连接池**：连接断开后，它的线程放回缓存，下个新连接优先复用缓存里的线程，缓存空了才新建线程。默认值 `-1` 表示自动调整，公式为 `8 + (max_connections / 100)`，上限 100——`max_connections = 151` 时自动值是 9。设为 0 则关闭缓存，每次连接都新建线程。
- **线程缓存命中率这样算**：`Threads_created` 是累计新建过的线程数，`Connections` 是累计连接尝试次数。命中率 ≈ `1 − Threads_created / Connections`。例如累计 100000 次连接、新建了 950 个线程，命中率约 99.05%。手册的口径是“对比 `Connections` 与 `Threads_created` 的差值看缓存效率”；“新建占比持续高于约 1% 就该警觉”是社区经验值（非手册规定，见来源）。

### 每连接缓冲算一算

先列默认值（MySQL 8.0 官方手册核实，单位字节）：

| 变量 | 默认值 | 用途 |
|---|---|---|
| `sort_buffer_size` | 262144（256 KiB） | 每个需要排序的会话分一份 |
| `join_buffer_size` | 262144（256 KiB） | 每个无索引 join 分一份，8.0.18+ 也管哈希 join |
| `read_buffer_size` | 131072（128 KiB） | 顺序扫描时每线程每表分一份 |
| `read_rnd_buffer_size` | 262144（256 KiB） | 排序后按序读行、MRR 时每客户端分一份 |
| `tmp_table_size` | 16777216（16 MiB） | 内部内存临时表的上限（每张表，不是每连接） |
| `max_heap_table_size` | 16777216（16 MiB） | MEMORY 表可长到的上限；与 `tmp_table_size` 取较小者约束内部临时表 |

四项每连接缓冲合计：256 + 256 + 128 + 256 = **896 KiB/连接**。理论上限换算（只计这四项，实际按需分配，见边界条件）：

| 并发连接数 | 四项缓冲理论上限 |
|---|---|
| 100 | 87.5 MiB |
| 500 | 437.5 MiB |
| 1000 | 875 MiB |
| 151（默认值打满） | 132.125 MiB |

临时表另计：`tmp_table_size` 是**每张**内部临时表的上限，不是每连接固定额度。极端假设（每个连接同时各持一张顶格 16 MiB 的内存临时表）才再加 100 连接 1.5625 GiB、500 连接 7.8125 GiB、1000 连接 15.625 GiB；151 连接打满时四项缓冲加临时表极端值合计约 2.49 GiB。注意 `tmp_table_size` 与 `max_heap_table_size` 对同一张 MEMORY 引擎内部临时表取较小者，**不叠加**；TempTable 引擎（8.0.28+ 内部临时表默认引擎之一）只看 `tmp_table_size`。

### 连接内存预算公式与 max_connections 设置建议

- **预算公式**：`max_connections ≤ (可用内存 − 全局缓冲) / 每连接缓冲上限`。全局缓冲的大头是 `innodb_buffer_pool_size`（本篇「小技巧」节已有“专用库服务器建议设到物理内存 70%~80%”）；每连接缓冲上限按上表 896 KiB/连接起算，再按业务里临时表的使用情况留余量。
- **算例（个人推断，仅作量级参考）**：8 GiB 内存机器，缓冲池占 70%（5.6 GiB），剩余约 2.4 GiB；2.4 GiB ÷ 896 KiB ≈ 2808，即四项缓冲的理论上限要到约 2808 个连接才会吃掉剩余内存。结论不是“可以设 2808”，而是**默认值 151 下每连接缓冲根本不是内存瓶颈**——先看 `Threads_running`、CPU 与锁，再考虑动 `max_connections`。
- **设置建议**：不要一上来就调大 `max_connections`。先上连接池（池大小按应用实例数 × 单实例池上限反推总连接数，别让 10 个实例 × 100 池直接冲垮 151 上限）；再确认空闲连接及时回收；最后才按预算公式调大，并用 `SET GLOBAL max_connections = N` 临时生效、`SET PERSIST` 或写配置文件持久化。调大前先确认有效上限还受 `open_files_limit − 810` 制约（手册原文），文件句柄不够时设再大也不生效。

### 连接风暴与排查口径

- **连接风暴的典型样子**：短连接（每次请求新建连接、用完即关）、连接池打满后新请求排队、空闲连接长期占位。空闲连接要等 `wait_timeout`（默认 28800 秒，即 8 小时，8.0.46 实测文章确认）才回收，所以“连接数高”先别怪并发高，先看是不是一堆 `Sleep` 状态的空闲连接没释放。
- **三步排查**：
  1. `SHOW STATUS LIKE 'Threads_connected'` 对比 `max_connections`：持续高于 80% 就告警，接近打满时先看下一步。
  2. `SHOW PROCESSLIST` 看状态分布：`Threads_running` 很小、`Sleep` 占大头 → 空闲连接占位或连接池泄漏，查应用侧池配置与连接归还逻辑；`Threads_running` 也高 → 真并发高或慢 SQL 堆积，先看慢查询（见本篇「慢查询排查速记」）。
  3. `SHOW STATUS LIKE 'Threads_created'` 对比 `Connections`：新建线程占比持续高于约 1%（社区经验阈值）→ 线程缓存太小或短连接太多，先上连接池减少建连次数，再考虑调大 `thread_cache_size`；同时看 `Aborted_connects` 是否在涨，涨说明有大量建连失败（网络、鉴权、打满被拒）。
- **线程缓存命中率低不是连接风暴的根因**：缓存只能省“新建线程”这点开销，连接握手与鉴权一次都省不了；短连接问题靠连接池解决，不是靠调大 `thread_cache_size`。

### 边界条件

1. **缓冲是按需分配，表中数字是理论上限、不是常驻占用**：空闲连接几乎不占这四项缓冲；只有真的排序、无索引 join、顺序扫描、随机读时才分配，不用的那一份就不占内存。
2. **按“每连接各一份”是简化口径**：`join_buffer_size` 是每个无索引 join 一份，多表 join 可能同时持有多份；`sort_buffer_size` 是每次排序一份；一条复杂 SQL 可能同时持有多份缓冲，表中数字是下限口径的上限估算，不是绝对最大值。
3. **`tmp_table_size` 是每张临时表的上限，且与 `max_heap_table_size` 不叠加**：MEMORY 引擎内部临时表受两者较小值约束，TempTable 引擎只看 `tmp_table_size`；多数连接根本不会建临时表，拿 16 MiB × 连接数当常驻开销算是错的。
4. **`thread_cache` 只复用线程、不复用连接**：新连接仍要走 TCP 握手与鉴权；命中率高只能省线程创建开销，救不了连接风暴，短连接必须靠连接池。
5. **`max_connections` 是上限、不是目标值**：设 1000 不等于能扛 1000 并发，瓶颈常在 CPU、锁与磁盘；有效上限还受 `open_files_limit − 810` 制约。
6. **`read_buffer_size` / `read_rnd_buffer_size` 在 InnoDB 为主的业务里很少触达**：前者主要面向 MyISAM 顺序扫描，后者面向 MyISAM 与 MRR；把这两项算满是保守上限，实际通常更低。
7. **版本差异待验证**：本次只核实了 MySQL 8.0 与 8.4 手册；5.7 的同名变量默认值未逐项核实。项目用 OceanBase MySQL 模式（兼容 5.7 语法），照搬前请以 `SHOW VARIABLES` 实测为准。
8. **OceanBase 差异待验证**：OceanBase MySQL 模式下连接与线程模型、上述变量是否生效，本次未核实，不展开。

> 来源（核验日期 2026-11-12）：MySQL 8.0 官方手册「Server System Variables」（`max_connections` 默认 151、`sort_buffer_size` / `join_buffer_size` / `read_rnd_buffer_size` 默认 262144、`read_buffer_size` 默认 131072、`tmp_table_size` / `max_heap_table_size` 默认 16777216、`thread_cache_size` 默认 -1 自动调整、公式 `8 + (max_connections / 100)` 上限 100）：https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html ；MySQL 8.4 官方手册同页核实 `max_connections` 默认 151、`max_heap_table_size` 默认 16777216：https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html ；MySQL 8.0 官方手册同页 `thread_cache_size` 条目（线程缓存语义：断开后线程回缓存、缓存空了才新建线程）；Percona《Dealing With "Too Many Connections" Error in MySQL 8》（默认 151、预留 1 个管理连接、ERROR 1040）：https://www.percona.com/blog/dealing-with-too-many-connections-error-in-mysql-8/ ；Netdata《MySQL connection exhaustion》（`Threads_created / Connections` 高于约 0.01 提示缓存太小或缺连接池、`Threads_connected / max_connections` 高于 0.8 告警，社区经验值）：https://www.netdata.cloud/guides/mysql/mysql-connection-exhaustion/ 。`wait_timeout` 默认 28800 秒（8 小时）据 8.0.46 实测文章，未在手册页逐项核对。8 GiB 算例与“先看 Threads_running 再动 max_connections”是个人推断；OceanBase 差异与 5.7 默认值差异待验证。

## 实战案例

### 案例一：join 选错叠加索引失效，一条关联查询越跑越慢

以下为设计演练示例，非真实生产事故。

- **现象**：一条两表关联查询，数据量不大时很快，数据量涨上去后突然变慢，慢查询日志里排进 Top SQL；单表查各自定义的索引都在，关联条件上也有索引，换写法、加缓存都不见效。
- **排查**：先开慢查询日志圈定这条 SQL，再用 `EXPLAIN` 看计划：type 掉到 `ALL`，key 为 NULL，Extra 出现 `Using filesort`；再看关联条件——一侧的关联列写了函数，另一侧是字符串列、查询时没加引号。怀疑优化器选错索引时，用 `EXPLAIN ANALYZE`（8.0.18+）核对实际行数与实际耗时，确认不是预估偏差。
- **根因**：两处索引失效叠加，优化器没得选，只能退回 BNL。函数运算与隐式转换都让关联列无法按索引键定位（见原文索引速记的失效速记）；被驱动表没有可用索引时，Nested Loop 每取一行就要全表扫一次，数据量一涨，扫表次数跟着涨；再叠加 `Using filesort`，排序又多花一份。单表索引“都在”，不等于关联时“能用上”。
- **修复**：去掉关联列上的函数，让条件写成能直接按索引键定位的形式；字符串列查询补上引号，消除隐式转换；确认被驱动表的关联列上有索引，缺失则补；仍有 `Using filesort` 且数据量大时，按排序键补复合索引，让排序变顺序读。改完再跑 `EXPLAIN`，确认 type 回到 `ref` / `range`、key 不再为 NULL、Extra 不再出现 `Using filesort`。
- **复盘**：join 慢先别急着换算法，先看被驱动表有没有可用索引——BNL 是缺索引的症状，不是 join 本身选错了。索引失效的排查顺序是固定的：先看条件写法（函数、隐式转换、`LIKE '%xxx'`、OR 一边没索引），再看优化器是不是判断全表更快，最后才考虑是不是统计信息或算法选择的问题。`EXPLAIN ANALYZE` 留到怀疑优化器选错时再用，日常排障先 `EXPLAIN` 看计划。

### 案例二：连接数打满，新连接报 Too many connections

以下为设计演练示例，非真实生产事故。

- **现象**：应用陆续报建连失败，报错为 `ERROR 1040 (08004): Too many connections`；已在跑的请求大多正常，慢 SQL 并不集中，新起的应用实例连不上库。
- **排查**：第一步，`SHOW STATUS LIKE 'Threads_connected'` 对比 `max_connections`：已持续高于 80%（社区经验告警线，非手册规定），接近打满。第二步，`SHOW PROCESSLIST` 看状态分布：`Threads_running` 很小，`Sleep` 占大头，说明不是真并发高，是空闲连接占位。第三步，`SHOW STATUS LIKE 'Threads_created'` 对比 `Connections`：新建线程占比持续高于约 1%（社区经验阈值，非手册规定），同时看 `Aborted_connects` 是否在涨。
- **根因**：短连接叠加连接池泄漏，空闲连接长期占位。连接断开后线程才回缓存，连接不断开，线程和会话内存就不释放；空闲连接要等 `wait_timeout`（默认 28800 秒，即 8 小时，8.0.46 实测文章确认）才回收。`max_connections` 默认 151（MySQL 8.0 与 8.4 官方手册均核实），多实例 × 大池的配置很容易在不知不觉中冲垮这个上限。线程缓存命中率低是同一批症状的另一面，不是根因——缓存只能省新建线程的开销，握手与鉴权一次都省不了。
- **修复**：先保住排障通道：持有 `CONNECTION_ADMIN`（或已废弃的 `SUPER`）权限的账号在打满后仍能连上，别把这个救命通道授给业务账号。再处理占位：查应用侧连接池配置与连接归还逻辑，确认空闲连接及时回收；短连接改走连接池，池大小按应用实例数 × 单实例池上限反推总连接数，别让多个实例的池子加起来冲垮 151 上限。最后才按预算公式考虑调大 `max_connections`：`max_connections ≤ (可用内存 − 全局缓冲) / 每连接缓冲上限`，每连接四项缓冲合计 896 KiB/连接（官方手册核实的默认值：`sort_buffer_size` / `join_buffer_size` / `read_rnd_buffer_size` 各 256 KiB，`read_buffer_size` 128 KiB），并用 `SET GLOBAL` 临时生效、`SET PERSIST` 或写配置文件持久化；调大前先确认有效上限还受 `open_files_limit − 810` 制约。
- **复盘**：连接数高先别怪并发高，先看 `Sleep` 占多少——空闲连接占位与真并发高是两种病，治法完全不同。不要一上来就调大 `max_connections`：默认值 151 下每连接缓冲根本不是内存瓶颈，先看 `Threads_running`、CPU 与锁，再考虑动上限；缓冲是按需分配的，空闲连接几乎不占这四项缓冲，拿满配去算常驻开销是错的。线程缓存只复用线程、不复用连接，短连接问题靠连接池解决，不是靠调大 `thread_cache_size`（默认值 -1 自动调整，`max_connections = 151` 时自动值是 9，公式 `8 + (max_connections / 100)`，上限 100）。

## 小技巧与冷知识

【小技巧】只看执行计划的预估不够，用 `EXPLAIN ANALYZE`（8.0.18+）看实际执行：它会真的跑一遍查询并打出每一步的实际耗时和实际行数，优化器“以为很快、实际很慢”的误判一眼就能看出来。线上排障先 `EXPLAIN` 看计划，怀疑优化器选错索引时再上 `EXPLAIN ANALYZE`，别一上来就对生产库跑它——它真的会执行那条 SQL。

【小技巧】`INT(11)` 里的 11 只是显示宽度，不限制存多大的数：`INT` 存的永远是 4 字节有符号整数，范围 -21 亿到 21 亿，写 `INT(1)` 也一样能存 20 亿。8.0.17 起整数的显示宽度已被废弃，建表时别再纠结括号里的数字，该纠结的是用 INT 还是 BIGINT。

【小技巧】`VARCHAR(255)` 这个 255 不是随便定的：长度小于 256 时长度前缀只占 1 个字节，超过 255 就要 2 个字节存长度，所以 255 是“单字节长度前缀能表示的最大值”。原文说“n 按实际设计，别无脑 255”是对的，补一个背景：255 本身没有性能魔法，短字段该用 32 就用 32。

【小技巧】想让写入幂等，用 `INSERT ... ON DUPLICATE KEY UPDATE` 而不是“先 SELECT 再决定 INSERT 还是 UPDATE”：主键或唯一键冲突时自动走 UPDATE 分支，一条语句完成“有则改、无则插”，还少了一次往返和并发下的竞态窗口。注意它只在唯一键冲突时触发，普通列重复不会触发，别拿它当去重工具。

【小技巧】`REPLACE INTO` 不是“更方便的 INSERT”，它是 DELETE 再 INSERT：先删旧行再插新行，DELETE 触发器会触发一次，自增 ID 也会重新分配，关联的外键行可能被级联影响。只想“冲突时更新几个字段”用 `ON DUPLICATE KEY UPDATE`，`REPLACE INTO` 留给“整行就是要被替换掉”的场景。

【小技巧】读一致性快照和加锁读是两回事：普通 `SELECT` 读的是事务开始时的快照（不加锁），`SELECT ... FOR UPDATE` 是对读到的行加排他锁、`LOCK IN SHARE MODE`（8.0 里叫 `FOR SHARE`）是加共享锁。扣库存、转账这类“读完马上要改”的逻辑必须用 `FOR UPDATE`，只用快照读会在高并发下算出两份扣减。

【小技巧】`CHECK` 约束在 5.7 里只解析、不执行，8.0.16 起才真正生效：5.7 建表写 `CHECK (amount >= 0)` 不会报错、也不会拦住负数入库，校验全靠应用层。项目还在 5.7/OceanBase 5.7 兼容模式时，别指望数据库替你拦住非法值；升到 8.0.16+ 之后它才是一道真防线。

【小技巧】MySQL 8.0 里 `GROUP BY` 不再隐式排序：5.7 及之前 `GROUP BY` 会顺带按分组列排好序，8.0 取消了这个隐式行为，依赖排序结果的代码必须显式加 `ORDER BY`。升级 8.0 后“分组结果顺序变了”不是数据乱了，是隐式排序被拿掉了，查这类问题先看 SQL 里有没有漏写 ORDER BY。

【小技巧】想精确区分大小写比对（如验证码、订单号），用 `utf8mb4_bin` 而不是默认的 `_ci` 排序规则：`utf8mb4_general_ci` 里 'AbC' 和 'abc' 算相等，`WHERE code = 'AbC'` 会把 'abc' 也查出来；`utf8mb4_bin` 按字节逐个比，大小写、全半角都分得清。只给需要精确比对的列单独设 `_bin`，别全库改——默认 `_ci` 对大多数业务查询更友好。

【小技巧】5.7 里第一列 `TIMESTAMP` 会悄悄带上默认值和自动更新：`DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` 是 5.7 的隐式行为，建表时没写也会有；8.0 默认 `explicit_defaults_for_timestamp=ON`，不再自动加。跨版本迁移时“更新时间怎么不自动变了”先查这个变量，而不是怀疑业务代码漏了赋值。

【小技巧】备份 InnoDB 表用 `mysqldump --single-transaction`：它在一次事务里导出一致快照，导出期间不锁表，业务照常读写。别对 InnoDB 表用 `--lock-tables`，那会把读也堵住；`--single-transaction` 只对 InnoDB 有效，混了 MyISAM 表的部分还是会被锁，导出前先确认引擎。

【小技巧】找“谁在拖慢库”先看 `SHOW PROCESSLIST` 再看慢日志：`SHOW PROCESSLIST` 能直接看到当前每个连接在跑什么 SQL、跑了多久、卡在哪个状态（如 `Sending data`、`Waiting for table metadata lock`），慢日志只记录已经跑完的。长事务、DDL 锁表这类正在发生的故障，PROCESSLIST 比日志快半拍。

【小技巧】`innodb_buffer_pool_size` 是最值得调的一个参数，专用数据库服务器上官方建议设到物理内存的 70%~80%：缓冲池是 InnoDB 的数据和索引缓存，太小了热点数据反复从磁盘读，太大又会和系统其他内存抢。改完要重启才生效，调之前先看 `SHOW STATUS LIKE 'Innodb_buffer_pool_reads'` 的磁盘读占比，确认真是缓冲池不够再动。

【冷知识】MySQL 的名字来自创始人 Monty Widenius 的大女儿 My，MariaDB 则来自小女儿 Maria：父女俩各占一个数据库的名字。MySQL 被 Sun 收购、再被 Oracle 收购后，Monty 另起炉灶做的 MariaDB 用了二女儿的名字，这段家族史也是 MySQL 和 MariaDB 分叉的起点。

【冷知识】自增计数器在 8.0 之前是存在内存里的，重启会回退：5.7 里重启 MySQL 后自增计数器会重置为 `MAX(id)+1`，删掉的最大 ID 会被重新用上；8.0 把计数器持久化到 redo log，重启后不再回退。原文说“自增不保证连续”在两个版本都成立，但 8.0 之后“重启复用已删 ID”这个坑已经没了。

【冷知识】`information_schema` 里没有一张真表：它全是内存里的视图，每次查询都是现场从各存储引擎收集元数据拼出来的。`SELECT * FROM information_schema.tables` 慢不是磁盘慢，是它在实时统计；查库表规模、找大表时直接查它没问题，但别在高频业务 SQL 里引用它。

【冷知识】MySQL 里也有 `DUAL` 这张虚拟表：`SELECT 1`、`SELECT NOW()` 这类不查任何表的语句，完整写法是 `SELECT 1 FROM DUAL`，DUAL 只是一行一列的占位表。从 Oracle 迁移过来的 SQL 里见到 `FROM DUAL` 不用删，MySQL 认它；自己写 SQL 时省略就行。

【冷知识】InnoDB 不是 MySQL 原生的引擎：它来自芬兰 Innobase 公司，2001 年左右被集成进 MySQL，2005 年 Innobase 被 Oracle 收购。MySQL 最早的默认引擎是 MyISAM（由 ISAM 演变而来），直到 5.5 才把默认引擎换成 InnoDB——“MySQL=事务+行锁”这个印象，其实是 InnoDB 后来带来的。

## 面试与自测

1. 深分页 `LIMIT 1000000, 20` 为什么越翻越慢？改成什么写法？
2. OceanBase MySQL 模式下，跨分区事务与单机 MySQL 有何不同？写代码时要注意什么？
3. 为什么说自增 ID 不能当业务序号用？分库分表下主键该用什么？

4. 线程缓存命中率怎么算？`Threads_created / Connections` 持续偏高说明什么？

### 参考答案

1. 越翻越慢是因为要先扫过前面 1000000 行再取 20 行。改游标分页：`WHERE id > 上次 id LIMIT 20`，用上次取到的最大 id 接着往后取。
2. 单分区内事务强一致，跨分区事务是弱隔离，写代码时别假设单机 MySQL 的锁行为；自增列行为也与单机不同（不保证连续），主键优先雪花 ID；部分系统表/函数不支持，迁移 SQL 先在 OB 测试环境跑一遍。
3. 自增 ID 是表级计数器，只保证本表单调递增，不保证连续——回滚、删除都会留空洞。分布式/分库分表下更不能依赖自增连续，各分片自增会冲突，应用雪花算法。

4. 线程缓存只复用断开连接后留下的空闲线程，命中看 `Threads_created/Connections`：新建线程占比越低越好，持续高于约 1% 说明缓存太小或短连接太多，应先上连接池、再调大 `thread_cache_size`。

## 关联知识

### 前置知识

- [数据库知识点地图](知识地图/数据库知识点地图.md)：关联，总览入口——先在地图里定位本篇在数据库知识体系中的位置，再决定往哪个专题篇深挖。
- [MySQL binlog](数据库/MySQL/MySQL-binlog.md)：关联，已拆分出去的专题——本篇只留入口，binlog 的格式与参数、主从复制、CDC 与 PITR 恢复都在该篇。
- [Redis](数据库/Redis/Redis.md)：关联，并列的另一类数据库——本篇的缓存与读写分离思路常与 Redis 配合落地，缓存本身的机制与选型见该篇。
- [Elasticsearch](数据库/ES/Elasticsearch.md)：关联，并列的另一类数据库——全文检索与分析型查询超出 MySQL 的舒适区时，对照该篇评估是否该换引擎。

### 延伸阅读

- [MySQL 索引与执行计划](数据库/MySQL/MySQL-索引与执行计划.md)：延伸，本节原理剖析中索引与执行计划部分的专题展开——索引设计口诀、EXPLAIN 完整读法与慢 SQL 闭环都在该篇。
- [MySQL 事务与 MVCC](数据库/MySQL/MySQL-事务与MVCC.md)：延伸，本节事务与锁部分的专题展开——事务与锁深挖见该篇。
- [慢查询诊断](数据库/MySQL/慢查询诊断.md)：延伸，案例一的排查方法论——慢查询日志圈定 Top SQL 后的完整诊断流程。
- [SQL 调优实战](数据库/MySQL/SQL调优实战.md)：延伸，案例一修复手法的实战对照——索引失效、回表与排序优化的更多实例。
- [锁等待与死锁分析](数据库/MySQL/锁等待与死锁分析.md)：延伸，案例二与死锁排查的专题展开——锁等待的定位与死锁成因的系统分析。
- [MySQL 主从复制与读写分离](数据库/MySQL/MySQL-主从复制与读写分离.md)：延伸，原文疑难杂症“主从延迟导致读到旧数据”的专题展开——复制延迟的成因与读写分离的取舍。
- [OceanBase](数据库/OceanBase.md)：延伸，原文 OceanBase 各节的完整展开——租户与资源单元、MySQL 模式兼容差异以该篇为准。
