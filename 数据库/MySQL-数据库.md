---
知识点: MySQL 数据库
层级: 进阶
适用版本: MySQL 5.7 / 8.x
更新日期: 2026-10-22
关联:
  - 数据库知识点地图
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

## OceanBase 租户与资源单元

```sql
-- OceanBase 的世界：集群 > 租户 > 数据库，和 MySQL 的"实例>库"不是一回事
SHOW TENANT;                          -- 看当前租户
SELECT * FROM oceanbase.DBA_OB_UNITS;  -- 资源单元（CPU/内存配额）
```

- **租户**是 OceanBase 的资源隔离单位：一个集群可以开多个租户（MySQL 租户、Oracle 租户），各租户 CPU/内存/磁盘配额独立。
- **资源单元（UNIT）**：租户的最小资源粒度，扩容 = 加 UNIT，不是加机器。
- 你在神农项目里连的是 MySQL 模式租户，`SHOW VARIABLES` 看到的是租户级变量，改 `ob_trx_timeout` 这类只影响当前租户。
- 排障先看租户资源：`__all_virtual_server_stat` 看 CPU/内存水位，满了先扩 UNIT 再查 SQL。

## 分库分表中间件选型

| 方案 | 类型 | 特点 |
|---|---|---|
| ShardingSphere-JDBC | 客户端 jar | 无额外运维，应用直连，适合中小规模 |
| ShardingSphere-Proxy | 代理 | 对应用透明（当 MySQL 用），可独立扩缩 |
| MyCat | 代理 | 老牌，社区活跃度下降，新项目不推荐 |

- 分片键选查询最常用的维度（用户 ID），别选时间（热点都挤在最新分片）。
- 能不分就不分：先读写分离 + 缓存，分表是最后手段（跨分片 join/事务是深渊）。
- 分了之后主键用雪花算法，别用自增（各分片自增会冲突）。

## binlog 基础与实战

### 是什么

- binlog（二进制日志）是 MySQL **Server 层**的逻辑日志，记录所有数据变更（DML 的实际改动 + DDL）。所有存储引擎通用，与引擎层的 redo log 不是一回事。
- 三大日志分工：

| 日志 | 层级 | 记录内容 | 用途 |
| --- | --- | --- | --- |
| binlog | Server 层 | 逻辑变更（语句或行镜像） | 主从复制、数据恢复、CDC 订阅 |
| redo log | InnoDB 层 | 页的物理变更 | 崩溃恢复（WAL） |
| undo log | InnoDB 层 | 修改前的数据版本 | 事务回滚、MVCC |

- 一致性靠**两阶段提交**：事务提交时先写 redo（prepare）→ 写 binlog → redo 标记 commit，保证 binlog 和 redo 要么都有要么都没有——主从数据一致、崩溃恢复不丢已提交事务的前提。
- 从库侧对应的是 **relay log**（中继日志）：从库 IO 线程把主库 binlog 拉回来落盘成 relay log，SQL 线程再重放。

### 三种格式

| 格式 | 记录内容 | 优点 | 问题 |
| --- | --- | --- | --- |
| STATEMENT | 原始 SQL 语句 | 体积小、可读 | `NOW()`、`UUID()`、无序 LIMIT 等不确定语句主从可能执行出不同结果 |
| ROW | 每行变更前/后镜像 | 主从绝对一致，可做闪回 | 体积大，SQL 本身不可见（需 mysqlbinlog 解码） |
| MIXED | 自动切换，遇不确定语句用 ROW | 折中 | 判断逻辑复杂，已基本淘汰 |

- MySQL 8.x 默认 ROW，新项目不要改。ROW 下还有 `binlog_row_image`：FULL（完整行镜像，默认，闪回靠它）/ MINIMAL（只记主键 + 变更列，省空间但闪回信息不足）/ NOBLOB。
- ROW 格式的事件链：`GTID_EVENT` → `QUERY_EVENT`(BEGIN) → `TABLE_MAP_EVENT` → `WRITE/UPDATE/DELETE_ROWS_EVENT` → `XID_EVENT`(COMMIT)。

### 关键参数（my.cnf demo，逐行注释）

```ini
[mysqld]
server_id = 1                          # 每台实例必须唯一，从库/Canal 伪装从库时也要有自己的 id
log_bin = mysql-bin                    # 开启 binlog，文件名前缀（mysql-bin.000001, 000002...）
binlog_format = ROW                    # 8.x 默认 ROW，不要用 STATEMENT
binlog_row_image = FULL                # 完整行镜像：闪回、CDC 拿变更前数据都靠它
sync_binlog = 1                        # 每个事务都刷盘 binlog；与 innodb_flush_log_at_trx_commit=1 合称“双1”，最安全但最慢
gtid_mode = ON                         # 开 GTID（全局事务ID：server_uuid:序号），主从切换免找位点
enforce_gtid_consistency = ON          # 开 GTID 必须同时开
binlog_expire_logs_seconds = 604800    # binlog 保留 7 天，过期自动删；太短会影响 PITR 恢复窗口
```

### 常用命令 demo

```sql
-- 查看 binlog 是否开启、格式、GTID
SHOW VARIABLES LIKE 'log_bin';          -- ON 才有 binlog
SHOW VARIABLES LIKE 'binlog_format';    -- 应为 ROW
SHOW MASTER STATUS;                     -- 当前正在写的 binlog 文件、位点、已执行 GTID 集合
SHOW BINARY LOGS;                       -- 所有 binlog 文件及大小
SHOW BINLOG EVENTS IN 'mysql-bin.000042' LIMIT 10;  -- 粗看事件（ROW 数据是 base64，看不懂细节）
```

```bash
# 解码 ROW 格式 binlog（运维/排查主力命令）
# --base64-output=decode-rows -v：把行事件解码成伪 SQL，### 开头的是变更前/后行数据
mysqlbinlog --base64-output=decode-rows -v mysql-bin.000042 | head -100

# 按时间窗口导出（定位误删、做 PITR 时用）
mysqlbinlog --start-datetime="2026-10-22 13:59:00" --stop-datetime="2026-10-22 14:01:00" \
  mysql-bin.000042 > /tmp/check.sql

# 远程拉取（binlog 不在本机时，加 --read-from-remote-server 和连接参数）
mysqlbinlog --read-from-remote-server -h 10.0.0.8 -u repl -p mysql-bin.000042 > /tmp/remote.sql
```

解码后 UPDATE 事件长这样（闪回和 CDC 都是按这个结构解析的）：

```text
### UPDATE `shop`.`orders`
### WHERE
###   @1=1001 /* id，主键，变更前 */
###   @2=2    /* status，变更前 */
### SET
###   @1=1001
###   @2=3    /* status，变更后 */
```

### 实际开发应用

**1. 主从复制**：binlog 是复制的数据源。从库 IO 线程拉 binlog → relay log → SQL 线程重放；8.0 支持多线程回放（按库/按逻辑时钟并行），延迟问题见下方「疑难杂症 3. 主从延迟」。开 GTID 后主从切换自动定位，不再手工算 file+position。

**2. CDC 增量同步（业务零侵入）**：订阅 binlog 把变更同步到 ES / Redis / 数仓 / 下游系统。

```bash
# canal.properties 关键（Canal 伪装成从库拉 binlog）
canal.serverMode = kafka               # 变更事件投递到 Kafka，也可直连 RocketMQ/TCP
canal.mq.topic = db_sync
```

- 场景：MySQL → ES（搜索）、MySQL → Redis（缓存更新）、MySQL → 数仓，业务代码零侵入。
- Canal 伪装成从库拉 binlog，挂了从上次位点续拉，不丢（但下游消费必须做幂等）。
- 典型链路：binlog → Canal → [RocketMQ](消息队列/RocketMQ.md) → 消费者更新 [Elasticsearch](数据库/Elasticsearch.md) / [Redis](数据库/Redis.md)。
- OceanBase 也有自己的增量同步（OMS），MySQL 那套照搬过去先看兼容性。
- 同类工具：Debezium（Kafka Connect 生态）、Flink CDC（流处理一体）。

**3. 误删恢复 / 按时间点恢复（PITR）**：全量备份只能回到备份时刻，中间的增量靠重放 binlog。

```bash
# 场景：14:00 误删 orders 表数据，14:02 发现；昨夜全量备份已恢复到临时库
# 1) 先解码确认误删语句的准确时间点（见上方解码命令），假设 DELETE 发生在 14:00:30
# 2) 把备份点之后、误删之前的 binlog 重放进临时库 —— 停在误删前一刻
mysqlbinlog --stop-datetime="2026-10-22 14:00:29" mysql-bin.000042 | mysql -h 临时库 -uroot -p
# 3) 从临时库把丢失数据导回生产（mysqldump 按条件导出或 INSERT INTO ... SELECT 跨实例）
```

- **闪回（flashback）**：ROW + FULL 镜像下，DELETE 事件里带着被删行的完整数据，反着生成 INSERT 即可回滚，不用全量重放。工具：binlog2sql（原理就是解码行事件后反向拼 SQL）。UPDATE 同理：WHERE 用变更后镜像、SET 用变更前镜像。
- 前提：误删之后 binlog 还没过期删除（看 `binlog_expire_logs_seconds`），所以保留期别设太短。

**4. Java 直接订阅 binlog demo**（mysql-binlog-connector-java，不依赖 Canal，适合自研小工具/审计）：

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.github.shyiko</groupId>
    <artifactId>mysql-binlog-connector-java</artifactId>
    <version>0.31.0</version>
</dependency>
```

```sql
-- 先建专用账号：只需要复制权限，别用 root
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl123';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';
```

```java
// 原理与 Canal 相同：把自己伪装成一个从库，连上主库后 binlog 事件会源源不断推过来
BinaryLogClient client = new BinaryLogClient("127.0.0.1", 3306, "repl", "repl123");
client.setServerId(1001); // 伪装从库的 server_id，必须和主库、其他从库都不同

// ROW 格式下先收到 TableMap（表结构映射），再收到行事件；
// connector 已把 tableId -> 表名 的映射处理好，这里只关心行变更
client.registerEventListener(event -> {
    EventData data = event.getData();
    if (data instanceof WriteRowsEventData) {          // INSERT
        WriteRowsEventData d = (WriteRowsEventData) data;
        d.getRows().forEach(row -> System.out.println("新增行: " + Arrays.toString(row)));
    } else if (data instanceof UpdateRowsEventData) { // UPDATE：每项是 (变更前, 变更后) 一对
        UpdateRowsEventData d = (UpdateRowsEventData) data;
        d.getRows().forEach(pair ->
            System.out.println("变更前: " + Arrays.toString(pair.getKey())
                             + " -> 变更后: " + Arrays.toString(pair.getValue())));
    } else if (data instanceof DeleteRowsEventData) { // DELETE：拿到的是被删行的完整镜像（FULL 时）
        DeleteRowsEventData d = (DeleteRowsEventData) data;
        d.getRows().forEach(row -> System.out.println("删除行: " + Arrays.toString(row)));
    }
});
client.connect(); // 阻塞式连接；断线后 connector 按最后位点自动续拉
```

- 生产注意：消费端必须幂等（位点是“至少一次”语义）；只订阅需要的库表（connector 支持按事件过滤，但 binlog 本身全量推送，量大时优先用 Canal + MQ 削峰）。

**5. 审计与变更追溯**：binlog 是全量变更的事实来源，可离线解析生成“谁在什么时候改了哪行数据”的审计流水（合规、扯皮对账时用）；注意 binlog 里没有“业务操作人”，要和应用日志按时间/事务对齐才有意义。

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
