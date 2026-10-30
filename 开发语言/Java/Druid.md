---
知识点: Druid 监控与慢 SQL 定位
层级: 进阶
适用版本: Druid 1.2.x
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】监控页里看到的 SQL 带问号占位符，是 StatFilter 默认把参数不同的同构 SQL 合并成一条统计：`druid.stat.mergeSql=true` 时，`WHERE id = 1` 和 `WHERE id = 2` 只算一条，执行次数和总耗时是汇总值。想按参数值分开看是哪条具体 SQL 慢，可以把 mergeSql 关掉，但统计条目会随参数值爆炸；平时定位慢 SQL 先看合并后的汇总，锁定某条再关合并下钻，别一上来就关。

【小技巧】只看总耗时会误判“最慢的 SQL”：执行次数多的 SQL 总耗时天然大，未必每次都慢。用 `TotalTime / ExecuteCount` 算平均耗时，筛出“每次都慢”的 SQL；再看 `MaxTimespan` 单次最大耗时抓偶发大慢查。原文第 3 节给的三个字段就是这么配合用的——总耗时看总量，平均耗时看稳定性，最大耗时看毛刺。

【小技巧】`test-while-idle` 只在空闲连接上探活，借出时并不校验：想借出时也验一次，用 `test-on-borrow: true`，但每次借连接都要多跑一次 `SELECT 1`，高并发下这笔开销不划算。生产上一般是空闲探活加合理的 eviction 间隔就够了；只有“偶发拿到坏连接、业务又极度敏感”的场景才开 test-on-borrow，权衡点在“每次借连接的成本”上。

【小技巧】池打满时别只看活跃连接数，还要看 `WaitThreadCount`：监控页连接池统计里，有线程在排队等连接时这个数大于 0。活跃连接长期满、又有线程排队，才是“池太小”；活跃连接满但没人排队，更可能是长事务占着连接（原文第 5 节的情况）。两个数一起看，才能分清是扩池还是修代码。

【小技巧】高频同构 SQL 可以开 PSCache 复用预编译语句：`pool-prepared-statements: true` 加 `max-pool-prepared-statement-per-connection-size`，每个连接缓存一批 PreparedStatement，MyBatis 每次执行的 `?` 占位符 SQL 不用反复编译。默认是关的——分库分表场景下 SQL 模板太多，缓存反而占内存；模板少、调用频次高的库开它收益才明显。

【小技巧】Wall 误杀时不用整个关防火墙，它可以逐项放行：Wall 配置里 `selectAllow`、`deleteAllow`、`updateAllow`、`insertAllow` 等是分开的开关，某类合法语句被拦，先只放开对应的那一项。原文第 4 节的 `log-violation` 是先观察再拦截的路子，逐项开关是观察之后的第二步——放行范围越小，防火墙剩下的保护就越多。

【小技巧】监控页除了 HTML，还有 JSON 接口可以直接拿数：`/druid/datasource.json`、`/druid/sql.json` 返回的就是监控数据，巡检脚本用 curl 拉 JSON 做阈值告警，比爬 HTML 监控页稳得多。原文第 3 节的 `DruidStatManagerFacade` 是代码里取数的路子，JSON 接口是脚本和外部监控取数的路子，两个入口按场景选，别只盯着浏览器页面。

【小技巧】filters 里除了 stat、wall，还能再挂一个日志 filter：`filter: stat,wall,slf4j` 这样配，SQL 执行日志会打进应用的 Slf4j 日志里，和慢日志互补——慢日志只记超阈值的，日志 filter 记的是全量执行。生产上全量 SQL 打日志量太大，一般只在排障时临时加上、用完摘掉，别长期挂着。

【冷知识】Wall 不是靠正则表达式拦注入的：它是基于 Druid 自带的 SQL Parser 做语义分析后拦截的，先把 SQL 解析成语法树再判断合不合法。所以注释里藏注入、`UNION` 拼接这类靠正则很难拦全的花样它能认出来；反过来，复杂合法 SQL 被误杀也是因为解析器是按语法判的——知道它是解析器，就明白误杀要从“语法”上找原因，而不是调正则。

【冷知识】Druid 不只是一个连接池，它是三件套：连接池（DruidDataSource）+ Filter 链插件体系 + 手写的 SQL Parser。StatFilter、WallFilter、日志 Filter 都是挂在 Filter 链上的插件，监控和防火墙不是池的附属功能，是同一套扩展机制长出来的；理解这一点就明白，为什么 Druid 能在“池”之外长出这么多监控和安全能力——扩展点本来就在设计里。

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
