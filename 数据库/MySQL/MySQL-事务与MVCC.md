---
知识点: MySQL 事务与 MVCC
层级: 进阶
适用版本: MySQL 5.7 / 8.x（InnoDB）
更新日期: 2026-10-23
关联:
  - MySQL-数据库
  - MySQL-索引与执行计划
  - MySQL-binlog
标签:
  - 数据库
  - 原理
  - 排错
---

# MySQL 事务与 MVCC

## 1. 核心概念

事务解决的是并发下的数据正确性：ACID 里原子性靠 undo log、持久性靠 redo log、隔离性靠锁 + MVCC 一起实现。理解事务，不要只背定义，要落到「这条数据现在有几个版本、谁能看见哪个」。

## 2. 隔离级别与可见性

| 级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
|---|---|---|---|---|
| 读未提交 RU | 有 | 有 | 有 | 几乎不用 |
| 读已提交 RC | 无 | 有 | 有 | 每条语句生成新快照；Oracle 默认 |
| 可重复读 RR | 无 | 无 | 基本无 | InnoDB 默认；快照读靠 MVCC、当前读靠间隙锁防幻读 |
| 串行化 | 无 | 无 | 无 | 读也加锁，并发最差，只在极端场景用 |

```sql
SELECT @@transaction_isolation;                 -- 看当前隔离级别（5.7 变量名 tx_isolation）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

关键区别只记一条：**RC 是每执行一条 SELECT 就刷新快照，RR 是事务第一次快照读时定格快照**。长事务在 RR 下会一直拿着旧快照，undo 版本链越攒越长，这就是「长事务拖垮数据库」的原理。

### 快照读 vs 当前读

- 快照读：普通 `SELECT`，读 MVCC 版本，不加锁。
- 当前读：`SELECT ... FOR UPDATE` / `LOCK IN SHARE MODE`、`UPDATE`、`DELETE`，读最新版本并加锁。
- 同一事务里先快照读、后当前读，会读到不同版本的数据，写代码时要有意识。

## 3. undo 与 redo

| 日志 | 作用 | 内容 |
|---|---|---|
| undo log | 回滚 + MVCC 版本链 | 修改前的数据，事务回滚时反向执行；旧版本靠它串成链 |
| redo log | 崩溃恢复、保证已提交不丢 | 页的物理修改，循环写；提交时先刷 redo（WAL），再异步刷数据页 |
| binlog | 主从复制、归档恢复 | 逻辑变更，server 层产生，见《MySQL-binlog》 |

两阶段提交（redo prepare → 写 binlog → redo commit）保证两个日志不打架，细节知道结论即可：**redo 决定事务算不算提交成功，binlog 必须与之对齐**。

## 4. MVCC 可见性判断

每行隐藏两个字段：`DB_TRX_ID`（最后修改的事务 ID）、`DB_ROLL_PTR`（指向 undo 里的旧版本）。快照读时拿当前事务的 ReadView 判断：

1. 版本的 trx_id 就是自己 → 可见。
2. trx_id 小于 ReadView 的最小活跃事务 ID（min_trx_id）→ 在快照生成前已提交，可见。
3. trx_id 大于等于最大事务 ID（max_trx_id）→ 快照之后才开启，不可见，沿版本链回溯。
4. 落在活跃列表里（还没提交）→ 不可见，回溯；不在列表里（已提交）→ 可见。

回溯就是沿 `DB_ROLL_PTR` 一版一版往旧了找，直到找到第一个可见版本。版本链太长时一次快照读要跳很多版——再次说明长事务是性能毒药。

```sql
-- 查正在跑的长事务（排障第一条）
SELECT trx_id, trx_started, trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx ORDER BY trx_started;

-- 查锁等待关系（8.0）
SELECT * FROM performance_schema.data_lock_waits;
```

## 5. 锁：行锁、间隙锁与死锁

- 记录锁（Record Lock）：锁索引记录本身。**InnoDB 的行锁锁的是索引**，没索引的更新会锁全表扫描过的所有行，效果近似表锁——更新语句必须走索引。
- 间隙锁（Gap Lock）：锁索引记录之间的空隙，只在 RR 下、当前读时生效，用来防幻读（插入新行）。
- 临键锁（Next-Key Lock）= 记录锁 + 间隙锁，RR 下当前读的默认形态。

```sql
-- 例：idx_status 上不存在 status=5 的行
BEGIN;
SELECT * FROM t_order WHERE status = 5 FOR UPDATE;  -- 锁住 status 所在的间隙
-- 此时其他事务 INSERT status=5 会被挡住（防幻读），但也更容易死锁
COMMIT;
```

### 死锁排查

```sql
SHOW ENGINE INNODB STATUS;   -- 看 LATEST DETECTED DEADLOCK 段：两个事务各持有什么锁、在等什么
```

死锁的成因永远是**加锁顺序不一致**或**锁范围过大**。治理顺序：

1. 业务层统一加锁顺序（如永远按 id 从小到大更新）。
2. 缩短事务：事务里不调远程接口、不做无关查询。
3. 保证更新走索引，避免单条 UPDATE 锁一片。
4. 应用层对死锁异常（1213）做有限重试——死锁是可恢复错误，重试是正规手段，不是掩盖问题。

## 6. 常见坑

- **隐式提交**：事务中执行 DDL（CREATE/ALTER 等）会自动提交前面的操作，迁移脚本里混 DDL 和 DML 要特别小心。
- **自增不回滚**：事务回滚后自增值不回收，ID 有空洞是正常现象，别拿自增当连续序号。
- **长事务不自知**：开了事务忘提交（连接池里连接被复用时更隐蔽）。监控 `innodb_trx` 里超过阈值的事务并告警。
- **`SELECT ... FOR UPDATE` 不带索引条件**：RR 下可能把整表范围都临键锁住，并发直接雪崩。
- **在 RC 下还假设不幻读**：项目改过隔离级别（或迁到 OceanBase）后，防幻读逻辑要重新评估，别按 RR 的直觉写代码。

## 7. 关联知识

- 锁等待与慢查询的交叉排查：[MySQL 数据库](数据库/MySQL/MySQL-数据库.md)
- 行锁锁的是索引，所以索引设计决定锁粒度：[MySQL 索引与执行计划](数据库/MySQL/MySQL-索引与执行计划.md)
- binlog 与两阶段提交：[MySQL binlog](数据库/MySQL/MySQL-binlog.md)
