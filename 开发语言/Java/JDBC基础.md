---
知识点: JDBC基础
层级: 基础
适用版本: 通用
更新日期: 2026-10-27
关联:
  - Druid
  - 多数据源
  - MyBatis原生
  - Java-异常体系
标签:
  - Java
  - 实践
---

# JDBC基础

## 1. 核心概念

JDBC（Java Database Connectivity）是 Java 访问关系型数据库的统一 API：业务代码只面向 `Connection` / `Statement` / `ResultSet` 这套接口写，具体数据库的驱动（MySQL Connector/J、OceanBase 驱动等）由厂商提供。理解 JDBC 的价值不在手写它——日常开发都用 MyBatis——而在理解 MyBatis、连接池、事务这些上层机制的底层行为：连接从哪来、事务边界在哪、资源什么时候还。

JDBC 的核心对象分工：

| 对象 | 职责 |
|---|---|
| `DataSource` | 连接的来源；生产一律用连接池提供的 DataSource，不用 DriverManager 直连 |
| `Connection` | 一次数据库会话；事务边界就画在它上面 |
| `Statement` / `PreparedStatement` | 执行 SQL 的载体；后者带占位符、防注入、可预编译 |
| `ResultSet` | 查询结果的游标，逐行向后读 |

## 2. 基础知识

### 完整流程：查一条记录

```java
public User findById(DataSource ds, long id) throws SQLException {
    String sql = "SELECT id, name, age FROM users WHERE id = ?";
    try (Connection conn = ds.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setLong(1, id);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                return new User(rs.getLong("id"), rs.getString("name"), rs.getInt("age"));
            }
            return null;
        }
    }
}
```

要点：

- `try-with-resources` 自动按「后开先关」顺序关闭 ResultSet → Statement → Connection，漏关一个就是连接泄漏（见第 5 节）。
- `ResultSet` 是游标：`next()` 移到下一行并返回是否有数据，`getXXX` 按列名或列序号取值；列序号从 1 开始，生产代码优先用列名，可读且不怕 SELECT 列顺序调整。
- `executeQuery` 跑 SELECT 返回 ResultSet；`executeUpdate` 跑 INSERT / UPDATE / DELETE 返回影响行数；两者混用直接抛异常。

### PreparedStatement：防注入 + 预编译

```java
// 错：字符串拼接，输入 "1 OR 1=1" 直接改变 SQL 语义
String sql = "SELECT * FROM users WHERE id = " + input;

// 对：占位符，输入永远只当参数值，不参与 SQL 解析
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setString(1, input);
```

- 占位符 `?` 从 1 开始编号，`setXXX` 的类型须与列类型匹配（`setLong` 对 bigint、`setString` 对 varchar），类型错在执行时才报。
- 预编译：同一 SQL 多次执行时数据库可复用执行计划；但「复用」是驱动与数据库配合的行为，别把它当成性能保证，批量场景看第 3 节批处理。

## 3. 进阶知识

### 事务控制：边界在 Connection 上

```java
try (Connection conn = ds.getConnection()) {
    conn.setAutoCommit(false);  // 关自动提交，手动控边界
    try {
        updateA(conn);
        updateB(conn);
        conn.commit();
    } catch (SQLException e) {
        conn.rollback();  // 任一步失败，整批回滚
        throw e;
    } finally {
        conn.setAutoCommit(true);  // 归还连接池前恢复，避免污染下一个借用者
    }
}
```

- 默认 `autoCommit = true`：每条 SQL 单独成事务，`updateA` 成功、`updateB` 失败时 A 已提交——多写操作必须关自动提交。
- 事务失效的经典来源：中途拿到的是另一个连接（多数据源下跨库、或方法里重新 `getConnection()`），两个写操作不在同一事务里；多数据源下的事务边界见 [多数据源](开发语言/Java/多数据源.md)。
- 隔离级别（`TRANSACTION_READ_COMMITTED` 等）由 `conn.setTransactionIsolation()` 设置，含义与选型见事务与 MVCC 相关笔记；JDBC 层面记住一点：隔离级别是连接级属性，改完同样要在归还前注意恢复。

### 批处理

```java
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement(
         "INSERT INTO users(name, age) VALUES (?, ?)")) {
    for (User u : users) {
        ps.setString(1, u.getName());
        ps.setInt(2, u.getAge());
        ps.addBatch();
    }
    ps.executeBatch();
}
```

- `addBatch` 只攒参数，`executeBatch` 一次发出；返回 `int[]` 是每条的更新计数，`Statement.EXECUTE_FAILED` 表示其中某条失败。
- 效果看驱动：以 MySQL Connector/J 为例，默认仍逐条发，批量拼成一条多值插入要靠驱动参数开启——批处理写了却没提速时，先查驱动参数，别怀疑批处理本身。
- 批处理与事务配合：大批写入时分段提交（如每 1000 条 commit 一次），避免单事务过大拖垮 undo / 锁。

## 4. 实战应用：与连接池 / Druid 的分工

业务代码的职责边界：只管「从 DataSource 拿连接、用完关连接」；连接的创建、复用、空闲回收、超时、泄漏检测全是池的事。

```java
// 业务代码只看到 DataSource，不知道背后是哪个池
public class UserDao {
    private final DataSource dataSource;

    public UserDao(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    // 方法内 getConnection() + try-with-resources，见第 2 节
}
```

- `conn.close()` 在池化场景下不是真关闭，而是**归还到池里**——所以「用完就关」在池化下没有性能损失，反而是正确用法。
- 池的参数（初始大小、最大活跃数、最大等待时间）与监控（活跃连接数、等待线程数）见 [Druid](开发语言/Java/Druid.md)；连接池被打满时先看 Druid 监控的活跃连接与等待队列，再怀疑慢 SQL。
- MyBatis 只是把第 2 节的手写流程（拼 SQL、设参数、映射 ResultSet）自动化了，底层走的仍是同一套 JDBC；MyBatis 的执行器与缓存见 [MyBatis原生](开发语言/Java/MyBatis/MyBatis原生.md)。

## 5. 疑难杂症

- **连接池打满、请求排队超时**：先查 Druid 监控的活跃连接数——持续高位说明有连接没归还，逐个检查 DAO 方法有没有漏 `try-with-resources`；再查有没有长事务占着连接不放（事务里调 HTTP、文件 IO）。
- **事务没回滚，脏数据已提交**：排查两点——异常被 catch 后没重抛也没调 `rollback()`（事务以为成功）；写操作走了另一个连接（见第 3 节）。
- **占位符与参数个数不一致**：`Parameter index out of range`，数 SQL 里的 `?` 个数与 `setXXX` 调用次数，动态拼 SQL 时最容易多一个或少一个。
- **ResultSet 关闭后取值**：`ResultSet closed`，取值必须在 ResultSet 关闭前完成；要把结果带出方法先映射成对象或集合，别把 ResultSet 本身返回。
- **中文乱码**：连接串的字符集参数与库表字符集不一致时出现，优先统一成 utf8mb4，见编码相关笔记。

## 6. 常见误区

- **Statement 拼字符串 SQL**：注入漏洞的头号来源，见第 2 节；搜索条件动态拼接时用 `PreparedStatement` + 条件累加参数列表。
- **循环里逐条查库（N+1）**：先查主表再逐条查关联表，数据量一上来就是几百次往返；改成 IN 批量查或 JOIN 一次取。
- **连接当成员变量长期持有**：连接不是线程安全的，且长期占用等于池里少一个连接；每次用每次拿、用完即关。
- **异常吞掉不回滚**：catch 里只记日志不 `rollback()`、不重抛，事务静默提交一半数据；异常处理规范见 [Java-异常体系](开发语言/Java/Java-异常体系.md)。
- **`SELECT *` 图省事**：列一多就拖带宽，表结构一变还可能打乱按序号取值的代码；只取需要的列。

## 7. 面试与自测

1. 为什么生产代码必须用 PreparedStatement 而不是 Statement 拼字符串？除防注入外还有什么收益？
2. 一段「先 update A 表、再 update B 表」的操作，A 成功、B 失败后 A 的数据还在，为什么？怎么改？
3. 连接池场景下 `conn.close()` 到底做了什么？为什么「用完就关」反而是正确用法？

### 参考答案

1. 占位符让输入只当参数值、不参与 SQL 解析，从根本上堵住注入；同时同一 SQL 可预编译复用执行计划。Statement 拼字符串既改变 SQL 语义，又每次都要重新解析。
2. 因为默认 `autoCommit = true`，每条 SQL 单独成事务，A 已提交。改法：关自动提交（`setAutoCommit(false)`），两条写操作走同一 Connection，成功 `commit`、失败 `rollback`，归还前恢复 `autoCommit = true`。
3. 池化场景下 `close()` 不是真关闭物理连接，而是归还到连接池供复用。所以「用完就关」没有建连开销，反而是及时归还、避免池被打满的正确用法；长期持有连接才是泄漏。

## 关联知识

- [Druid](开发语言/Java/Druid.md)
- [多数据源](开发语言/Java/多数据源.md)
- [MyBatis原生](开发语言/Java/MyBatis/MyBatis原生.md)
- [Java-异常体系](开发语言/Java/Java-异常体系.md)
