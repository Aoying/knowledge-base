---
知识点: Druid 监控与慢 SQL 定位
层级: 进阶
适用版本: Druid 1.2.x
更新日期: 2026-10-28
关联:
  - Java
  - MyBatis-Plus
  - MySQL-数据库
  - Spring-Boot
标签:
  - Java
  - 数据库
  - 实践
---

# Druid 监控与慢 SQL 定位

## 1. 核心概念

Druid 是阿里开源的 JDBC 连接池，自带两样别的池（HikariCP 等）没有的东西：**SQL 监控统计**（StatFilter：每条 SQL 的执行次数、总耗时、慢 SQL、最大并发）和 **Wall 防火墙**（WallFilter：按 SQL 语义防注入、防误删全表）。连接池本身的取舍是「功能全、监控强」对「极致轻量」；在需要线上定位慢 SQL 的企业项目里，监控能力的价值通常压过那点性能差（经验）。

## 2. 关键配置

```yaml
# application.yml
spring:
  datasource:
    druid:
      initial-size: 5          # 初始连接数
      min-idle: 5              # 最小空闲
      max-active: 50           # 最大活跃连接——按库能承受的并发定，不是越大越好
      max-wait: 60000          # 拿不到连接的最大等待(ms)，超时抛 GetConnectionTimeoutException
      validation-query: SELECT 1
      test-while-idle: true    # 空闲时探活，防数据库/防火墙掐掉空闲连接后的"假死连接"
      time-between-eviction-runs-millis: 60000
      filter:
        stat: { enabled: true, slow-sql-millis: 1000, log-slow-sql: true }
        wall: { enabled: true }   # 防火墙：防注入
```

池参数的定法：`max-active` 要和下游 DB 的连接上限、实例数一起算——3 个实例各 50 就是 150 个连接顶到库上，先确认库侧上限再定，别单机拍脑袋。`max-wait` 设有限值是为了让故障时快速失败、暴露问题；设成 -1 无限等会把故障拖成雪崩。

## 3. 监控页与慢 SQL 定位

- 浏览器开 `/druid/index.html` 看 SQL 监控页：执行次数、慢 SQL、max 并发，一目了然（生产记得加账号密码）。
- 慢 SQL 定位链路：druid 慢日志 → `EXPLAIN` 看执行计划 → 索引/分页优化 → 回 druid 验证耗时下降。
- 神农这种金融项目：`slow-sql-millis` 设 500ms，定时扫监控页，比等用户投诉强。阈值不是越小越好：太小日志会被正常波动淹没，看自己业务的耗时分布定（经验）。
- 监控页重点看三列：**执行次数**（揪出 N+1 式的高频小查询）、**最大耗时**（揪出偶发大慢查）、**读取行数/更新行数**（扫描行数远大于返回行数 = 没走好索引）。

```java
// 代码里拿慢 SQL TopN（做定时巡检/告警时用）
DruidStatManagerFacade facade = DruidStatManagerFacade.getInstance();
List<Map<String, Object>> stats = facade.getDataSourceStatDataList();
// 每项含 SQL、ExecuteCount、TotalTime、MaxTimespan 等字段，按需排序取 TopN
```

Boot 2.7 集成用 `druid-spring-boot-starter`，监控页账号密码走 `spring.datasource.druid.stat-view-servlet.login-username/login-password` 配置，并用 `reset-enable: false` 关掉页面上的重置统计按钮。

## 4. Wall 防火墙的坑

- Wall 是按语法分析拦截的，误杀真实存在：复杂 SQL（多层嵌套、特殊函数、`ON DUPLICATE KEY`、批量语句）可能被判违规。上线前在测试环境把核心 SQL 跑一遍 Wall，或者先开 `log-violation` 只记录不拦截，观察一段时间再转拦截。
- OceanBase（MySQL 模式）下个别语法与 MySQL 有细微差异，Wall 报错时先确认是真违规还是方言差异，别直接关防火墙了事。
- Wall 拦截抛 `WallFilterCheckException`，全局异常处理里要给它一个明确的错误码，别混进「系统异常」里看不见。

## 5. 连接泄漏排查

连接池打满是最常见的 Druid 线上事故：表现是 `max-wait` 超时、监控页活跃连接长期不降。排查顺序：

1. 先看监控页「连接池」与 SQL 统计，确认是真慢（某条 SQL 占着连接久）还是连接没还（业务代码拿了连接不关——多见于手写 JDBC 或流式查询没关流）。
2. 开 `remove-abandoned: true` + `remove-abandoned-timeout` 只能作临时止血和定位手段（它会在日志里打印疑似泄漏的堆栈），根因修掉后建议关掉，自动回收掩盖问题。
3. 事务方法里调了慢的外部接口（HTTP 等待几十秒），事务全程占着连接——这类「长事务占连接」在监控页上看就是活跃连接高、SQL 却不慢，要把外部调用移出事务。

## 6. 面试与自测

1. `max-active` 怎么定？为什么不能越大越好？`max-wait` 设成 -1 有什么后果？
2. 监控页活跃连接长期不降、但 SQL 统计并不慢，可能是什么问题？排查顺序是什么？
3. Wall 防火墙误杀真实 SQL 时怎么办？为什么不能直接关防火墙了事？

### 参考答案

1. `max-active` 要和下游库的连接上限、实例数一起算：3 个实例各配 50，就是 150 个连接顶到库上，先确认库侧上限再定，单机拍脑袋越大越容易把库打满。`max-wait` 设有限值是为了故障时快速失败、暴露问题；设成 -1 无限等，会把故障拖成雪崩。
2. 先怀疑「长事务占连接」而不是慢 SQL：事务方法里调了慢的外部接口（HTTP 等待几十秒），事务全程占着连接，监控页上看就是活跃连接高、SQL 却不慢。排查顺序：监控页确认连接与 SQL 统计 → 区分真慢还是连接没还（手写 JDBC/流式查询没关流）→ 把外部调用移出事务。`remove-abandoned` 只作临时止血和定位（日志里打印疑似泄漏堆栈），根因修掉后建议关掉，自动回收会掩盖问题。
3. 先在测试环境把核心 SQL 跑一遍 Wall，或先开 `log-violation` 只记录不拦截、观察一段时间再转拦截；OceanBase（MySQL 模式）下先确认是真违规还是方言差异。直接关防火墙等于丢掉防注入、防误删全表那层保护。Wall 拦截抛 `WallFilterCheckException`，全局异常处理里要给它一个明确的错误码，别混进「系统异常」里看不见。

## 关联知识

- [多数据源（dynamic-datasource）](开发语言/Java/多数据源.md)
- [MyBatis-Plus](开发语言/Java/MyBatis/MyBatis-Plus.md)
- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
