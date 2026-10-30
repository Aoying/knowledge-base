---
知识点: MySQL 数据库
层级: 进阶
适用版本: MySQL 5.7 / 8.x
更新日期: 2026-10-31
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

### 参考答案

1. 越翻越慢是因为要先扫过前面 1000000 行再取 20 行。改游标分页：`WHERE id > 上次 id LIMIT 20`，用上次取到的最大 id 接着往后取。
2. 单分区内事务强一致，跨分区事务是弱隔离，写代码时别假设单机 MySQL 的锁行为；自增列行为也与单机不同（不保证连续），主键优先雪花 ID；部分系统表/函数不支持，迁移 SQL 先在 OB 测试环境跑一遍。
3. 自增 ID 是表级计数器，只保证本表单调递增，不保证连续——回滚、删除都会留空洞。分布式/分库分表下更不能依赖自增连续，各分片自增会冲突，应用雪花算法。
