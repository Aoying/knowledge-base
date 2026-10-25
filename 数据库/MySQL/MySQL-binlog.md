---
知识点: MySQL binlog
层级: 进阶
适用版本: MySQL 5.7 / 8.x
更新日期: 2026-10-25
关联:
  - MySQL-数据库
  - RocketMQ
  - 数据库知识点地图
标签:
  - 数据库
  - 实践
---

# MySQL binlog

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
| ROW | 每行变更前/后镜像 | 主从基本一致（自增列、时区、触发器等边界仍有例外），可做闪回 | 体积大，SQL 本身不可见（需 mysqlbinlog 解码） |
| MIXED | 自动切换，遇不确定语句用 ROW | 折中 | 判断逻辑复杂；8.0 仍支持，并未淘汰，新项目默认 ROW，不必选它 |

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
SHOW BINARY LOG SOURCE STATUS;          -- 当前正在写的 binlog 文件、位点、已执行 GTID 集合（8.0.22 起替代 SHOW MASTER STATUS，8.4 已移除后者）
SHOW REPLICA STATUS;                      -- 从库侧看复制状态用这条（8.0.22 前叫 SHOW SLAVE STATUS）
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

**1. 主从复制**：binlog 是复制的数据源。从库 IO 线程拉 binlog → relay log → SQL 线程重放；8.0 支持多线程回放（按库/按逻辑时钟并行），延迟成因与对策见 [MySQL-主从复制与读写分离](数据库/MySQL/MySQL-主从复制与读写分离.md) 第 4 节；binlog 侧的延迟判断见下方「主从延迟 binlog 侧」一节。开 GTID 后主从切换自动定位，不再手工算 file+position。

**2. CDC 增量同步（业务零侵入）**：订阅 binlog 把变更同步到 ES / Redis / 数仓 / 下游系统。

```bash
# canal.properties 关键（Canal 伪装成从库拉 binlog）
canal.serverMode = kafka               # 变更事件投递到 Kafka，也可直连 RocketMQ/TCP
canal.mq.topic = db_sync
```

- 场景：MySQL → ES（搜索）、MySQL → Redis（缓存更新）、MySQL → 数仓，业务代码零侵入。
- Canal 伪装成从库拉 binlog，挂了从上次位点续拉，不丢（前提是位点还在且对应 binlog 未被 purge；位点丢了或 binlog 已过期删除，续拉就断档，只能重做快照；下游消费必须做幂等）。
- 典型链路：binlog → Canal → [RocketMQ](消息队列/RocketMQ.md) → 消费者更新 [Elasticsearch](数据库/Elasticsearch.md) / [Redis](数据库/Redis/Redis.md)。
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
    <groupId>com.zendesk</groupId>
    <artifactId>mysql-binlog-connector-java</artifactId>
    <version>0.30.3</version>
</dependency>
```

- 坐标说明：shyiko 原库最后版本是 0.21.0（2020-06），之后维护在 osheroff fork（com.zendesk 坐标，Maven Central 已核实最新为 0.30.3，2026-01-05）；osheroff 仓库有 v0.31.0 标签（2026-07），但未核实 Central 发布，暂不用。包名仍是 com.github.shyiko.mysql.binlog.*，代码不用改。原文 com.github.shyiko:0.31.0 无发布记录，不要用。

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

## 生产纵深补齐

### 组提交与刷盘策略全谱

- `sync_binlog` 三档：0 = 只写 OS 缓存、由 OS 刷盘，崩溃可能丢已提交事务；1 = 每事务 fsync，最安全、最慢；N = 每 N 组事务 fsync 一次，折中。文中示例用 1，与 `innodb_flush_log_at_trx_commit=1` 合称“双 1”。
- 组提交：`binlog_group_commit_sync_delay`（微秒级等待，把多次提交合并成一次 fsync）+ `binlog_group_commit_sync_no_delay_count`（攒够多少事务不等了直接刷），用延迟换吞吐；延迟敏感业务别把 delay 开大。
- 关键风险：两阶段提交下 redo 先 prepare 再写 binlog；`sync_binlog=0` 时崩溃可能出现 redo 已提交、binlog 未落盘——主库有这条事务、从库没有，主从静默不一致。只放宽 sync_binlog、不放宽 redo 侧，丢的是 binlog 不是数据，但复制关系会因此断档。

### 保留时长与磁盘风险

- 8.0 默认 `binlog_expire_logs_seconds=2592000`（30 天），过期自动 purge；文中示例 604800（7 天）只是示例，保留期要按 PITR 窗口倒推，别照抄。
- `max_binlog_size`（默认 1GB）只是单个文件上限，不是总量上限；总量 ≈ 保留期 × 写入速率，大库很容易超出预期。
- 磁盘打满的后果：binlog 写不下去，事务无法提交，主库停写。监控水位三件套：磁盘使用率、最早 binlog 文件时间、`SHOW BINARY LOGS` 的总大小；保留期缩短前先确认 PITR 窗口还能接受。

### 大事务 / 长事务危害

- 事务提交时才一次性写 binlog，提交前事件攒在 binlog cache 里：大事务会让 cache 溢出到磁盘临时文件（`binlog_cache_size` 默认 32KB，超了就落盘），提交瞬间产生大 IO。
- 从库侧：一个大事务要一口气回放完才推进位点，延迟陡增；CDC 下游会一次性收到巨量事件，下游容易被打爆。
- 治理：批量 DML 拆小事务分批提交、长事务及时提交；先治源头，再谈加从库或并行回放。

### GTID 纵深

- `gtid_executed` 是本机已执行 GTID 集合，排障先看主从差集，别只看 file+position。
- PURGE 风险：已执行 GTID 对应的 binlog 被 purge 后，从库再接入可能报 1236（找不到位点/GTID），只能重做快照或换位点。
- 孤儿事务：主库有、从库缺的 GTID，切换主库（CHANGE MASTER 到新主）时可能冲突，切换前先对齐 GTID 集合。
- `RESET MASTER` 会清空本机 GTID 集合与全部 binlog，是高危操作，只在重建复制关系时用，用前先确认没有从库还指着它。

### ROW 体积与压缩

- ROW 下批量更新、大字段行的体积膨胀最明显；先看 `binlog_row_image`：FULL 保闪回与 CDC 旧镜像，MINIMAL 省空间但闪回信息不足、CDC 拿不到完整旧镜像，两者取舍见文中三种格式一节。
- 8.0.20+ 事务压缩：`binlog_transaction_compression=ON`（+ zstd）把整个事务压成 `TRANSACTION_PAYLOAD` 事件，CPU 换空间与带宽；压缩率看 workload，OLTP 小事务收益有限，大事务收益明显。
- JSON 列部分更新：8.0 支持 JSON 列只记变更部分，能省体积，但仍要关注单行过大时的复制与回放成本。

### CDC 生产要点

- 位点管理：位点（文件+position 或 GTID 集合）要持久化到外部存储，重启续拉；位点推进与下游落库尽量放同一原子操作，否则不是重复就是丢失——位点语义是“至少一次”，下游必须幂等。
- DDL 与 schema history：表结构变了，旧事件按旧结构解析会错位；Debezium/Canal 都要配 schema 适配（schema history 主题或 Canal 的 schema 同步），DDL 变更后先验证消费端解析再放量。
- 延迟监控：打点源库写入时间 vs 消费端处理时间，别只看队列长度；队列空不代表无延迟（消费端卡住时队列也可能是空的）。
- 跨表顺序：单表内事件有序，多表之间不保证全局顺序，别跨表拼事务；需要跨表一致性的场景走下游对账而不是依赖 binlog 顺序。

### ROW 审计盲区

- ROW 事件只有行镜像，没有原始 SQL 文本；也没有“业务操作人”（哪个用户、哪个接口触发的），单靠 binlog 做不到“谁改的”。
- MINIMAL 下变更前镜像不全，审计更残；FULL 是审计的前提，不是可选项。
- 补法：应用层审计日志（操作人 + 接口 + 业务主键 + 时间）+ binlog 按时间/事务对齐；别为审计切回 STATEMENT——8.0 默认 ROW，且不确定语句在 STATEMENT 下有主从不一致风险。

### 闪回与 PITR 补全

- 定位优先用 `--stop-position` / GTID 集合，不用秒级 datetime：同一秒多个事务、时钟 skew 都会让 datetime 边界切错，误删边缘的一条事务最容易漏。
- 跨多个 binlog 文件时按文件顺序重放，别只导一个文件；恢复前先停写保护（只读或停业务写），防止恢复期间新写入又被覆盖。
- 恢复后必校验：行数 + 抽样 + 校验和，三者都对才算恢复完成；只看“命令跑完了”等于没验。
- 定期演练：备份 + binlog 恢复链路每季度至少跑一次，没演练过的恢复链路等于不存在；异地备份要和 binlog 备份对齐策略，否则异地恢复时缺增量。

### 主从延迟 binlog 侧

- 判断 relay 堆积：从库 `SHOW REPLICA STATUS` 看 `Relay_Log_File` / `Relay_Log_Pos` 与 `Master_Log_File` / `Exec_Master_Log_Pos` 的差；`Seconds_Behind_Source` 只是粗略口径（无流量时恒为 0），准确延迟看 GTID 差或心跳表打点。
- relay 相关参数：`relay_log_space_limit`（relay 总量上限，超了 IO 线程暂停拉取，防从库磁盘打满）；`relay_log_purge=ON`（自动清理已回放 relay）；`relay_log_recovery=ON`（崩溃后从主库重新拉 relay，防 relay 损坏导致回放错位）。
- 并行回放：`replica_parallel_workers`（8.0 按 WriteSet 并行，行不冲突即可并行回放）；workers 数按 CPU 与写冲突率调，不是越大越好，冲突高的 workload 开多 workers 也并行不起来。
- 延迟根因先看回放段：成因与对策见 [MySQL-主从复制与读写分离](数据库/MySQL/MySQL-主从复制与读写分离.md) 第 4 节。

### MySQL 8.0 / 8.4 版本注记

- `expire_logs_days` 在 8.0 已废弃，统一用 `binlog_expire_logs_seconds`；旧脚本里写前者的要改。
- `SHOW MASTER STATUS` / `SHOW SLAVE STATUS` 在 8.0.22 起弃用、8.4 已移除，改 `SHOW BINARY LOG SOURCE STATUS` / `SHOW REPLICA STATUS`（见常用命令一节）。
- 8.0 默认 `binlog_format=ROW`、默认按 WriteSet 并行回放；8.0.20+ 支持事务压缩（见 ROW 体积一节）；GTID 默认仍是 OFF，开 GTID 要显式设 `gtid_mode=ON` + `enforce_gtid_consistency=ON`。

## 实际应用场景与 demo 展示

本地可跑演示：Docker 起 MySQL 8.0、开 binlog、写几行数据、解码查看、GTID 定位恢复、最小 CDC 思路验证。端口用 3307，避免与本机 MySQL 冲突；演示完 `docker rm -f` 清理。

```ini
# /tmp/binlog-demo/my.cnf（挂载到 /etc/mysql/conf.d）
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_expire_logs_seconds = 604800
```

```bash
# 1) 起库
docker run --name mysql-binlog-demo -e MYSQL_ROOT_PASSWORD=demo123 \
  -p 3307:3306 -v /tmp/binlog-demo/my.cnf:/etc/mysql/conf.d/my.cnf -d mysql:8.0

# 2) 写几行数据
mysql -h 127.0.0.1 -P 3307 -uroot -pdemo123 -e \
  "CREATE DATABASE demo; USE demo; \
   CREATE TABLE t(id INT PRIMARY KEY, name VARCHAR(20)); \
   INSERT INTO t VALUES(1,'a'),(2,'b'); \
   UPDATE t SET name='a2' WHERE id=1; DELETE FROM t WHERE id=2;"

# 3) 看当前 binlog 与 GTID 集合（记下误删前的 GTID，后面恢复用）
mysql -h 127.0.0.1 -P 3307 -uroot -pdemo123 -e "SHOW BINARY LOG SOURCE STATUS\G"

# 4) 解码查看：应看到 TABLE_MAP + WRITE_ROWS / UPDATE_ROWS / DELETE_ROWS，DELETE 行带完整旧镜像
mysqlbinlog --read-from-remote-server -h 127.0.0.1 -P 3307 -uroot -pdemo123 \
  --base64-output=decode-rows -v mysql-bin.000001 | grep -A6 'UPDATE\|DELETE'
```

验证点：

- 解码输出里 `### UPDATE` / `### DELETE` 的 WHERE / SET 行与第 2 步的写操作一一对应，DELETE 的旧镜像就是被删的 `(2,'b')`。
- GTID 定位恢复：误删后用 `mysqlbinlog --exclude-gtids='<误删事务的 GTID>'` 重放，或用 `--stop-position` 停在误删前；恢复后 `SELECT COUNT(*)` 与误删前一致才算通。
- 最小 CDC 思路验证：跑文中第 4 节的 connector demo（server_id 换一个不冲突的值、账号用 repl 专用账号），控制台应依次打印新增行、变更前/后、删除行三类事件；只看到一类或顺序错乱，先查 binlog_row_image 是不是 FULL。

```bash
# 清理
docker rm -f mysql-binlog-demo
