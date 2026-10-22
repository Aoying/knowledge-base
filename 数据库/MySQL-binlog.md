---
知识点: MySQL binlog
层级: 进阶
适用版本: MySQL 5.7 / 8.x
更新日期: 2026-10-22
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

