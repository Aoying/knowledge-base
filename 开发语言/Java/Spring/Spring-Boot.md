---
知识点: Spring Boot
层级: 进阶
适用版本: Spring Boot 2.7.3
更新日期: 2026-10-31
关联:
  - Java
  - MyBatis-Plus
  - Druid
  - 多数据源
标签:
  - Java
  - 实践
---

# Spring Boot

## 1. 核心概念

Spring Boot 的价值是「约定优于配置」：自动配置（AutoConfiguration）根据 classpath 上的依赖与已有 Bean，推断并装配默认配置——有 Druid 就配 Druid 数据源、有 Web 依赖就起内嵌 Tomcat。理解自动配置的排查手段比背配置项重要：启动加 `--debug` 看 Condition Evaluation Report（哪些自动配置生效/没生效及原因），或看 `actuator/conditions` 端点。自己定义的 Bean 会让对应自动配置退让（`@ConditionalOnMissingBean`），这是「我明明配了却没生效/没配却生效了」类问题的主线。

起步依赖（starter）是预配好的依赖包：版本由 Boot 的 BOM 统一管理，别自己拼单个 jar 的版本；第三方库（MyBatis-Plus、Druid、Flowable）要查它们声明支持的 Boot 版本线。

## 2. 配置加载顺序（后覆盖前）

1. 命令行参数 `--server.port=8081`
2. `SPRING_APPLICATION_JSON` / 系统环境变量
3. `application-{profile}.yml` → `application.yml`

多环境用 profile 分文件：`application.yml` 只放公共项，`application-{env}.yml` 放差异项，靠 `spring.profiles.active`（或环境变量 `SPRING_PROFILES_ACTIVE`）切换。

两个高频坑：

- 同一目录下 `application.properties` 优先级高于 `application.yml`，项目里两种格式混用时改了 yml 不生效先查有没有同名 properties 在盖它。
- `@ConfigurationProperties` 绑定复杂配置（前缀 + 嵌套对象）比到处 `@Value` 好维护，且能配合 `@Validated` 在启动期就校验配置合法性；配置类记得加 `@EnableConfigurationProperties` 或标 `@ConfigurationPropertiesScan`（2.6+ 可用）。

### 常用配置片段

```yaml
server:
  port: 8080
  tomcat:
    max-threads: 200
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initial-size: 5
      max-active: 50
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 本地看 SQL
  global-config:
    db-config:
      logic-delete-value: 1
      logic-not-delete-value: 0
```

`log-impl` 打 SQL 只在本地开：生产用 StdOutImpl 会刷爆日志且拖慢吞吐，生产看 SQL 走 Druid 监控，见 [Druid](开发语言/Java/Druid.md)。

## 3. 事务：@Transactional 失效的 4 个经典场景

1. **同类自调用**：`this` 直接调，绕过代理。解法：注入自己 / `AopContext.currentProxy()`。
2. **private / final 方法**：代理不了，不生效。
3. **异常被吞**：try-catch 吃掉异常且没抛，事务以为成功。想回滚就 `throw new RuntimeException` 或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
4. **checked 异常**：默认只回滚 RuntimeException。受检异常要回滚加 `rollbackFor = Exception.class`。

补充两个同样常见的：**方法不是 public**（同第 2 条本质，CGLIB 代理要求 public）、**类没被 Spring 管理**（自己 new 出来的对象上的注解全都不生效，事务/异步/缓存注解同理）。

### 事务传播常用组合

- `REQUIRED`（默认）：有就加入，没有新建。
- `REQUIRES_NEW`：挂起当前，永远新建——记流水、审计日志独立提交用它。
- 注意：`REQUIRES_NEW` 内部抛异常不影响外部已提交的事务（已挂起隔离）。
- 另两个实战会用到的：`NESTED` 靠保存点实现「内层回滚不牵连外层」，批处理逐条容错时好用；`NOT_SUPPORTED` 非事务方式跑并挂起已有事务，大查询不占事务资源时用。

事务边界经验：事务标在 Service 层、只包一段明确的写操作；事务里别调 HTTP、别做文件 IO——长事务占着连接不放，是连接池被打满的头号来源，与 [多数据源](开发语言/Java/多数据源.md) 的跨库事务注意点一起看。

## 4. Actuator 与优雅停机

- Actuator 至少开 `health`、`info`、`metrics`；`health` 给探活用（K8s/负载均衡探活打 `/actuator/health`，并开 `show-details` 的权限控制），别把所有端点裸奔暴露在公网，`env`、`heapdump` 这类端点尤其危险，要么关、要么放内网管理端口（`management.server.port` 单独设）。
- 优雅停机：`server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`，让在途请求与异步任务有收尾时间；与 @Async 的停机等待配合，见 [@Async 异步](开发语言/Java/Async-异步.md)。
- 日志：Boot 默认 Logback，yml 里用 `logging.level.*` 调级别；线上临时调级别可用 Actuator 的 `loggers` 端点（POST 改、DELETE 恢复），比重启现实，同样要做访问控制。

## 5. Spring Boot 2.7 → 升级路线图（2026-10-20 核验）

| 线 | 状态 |
|---|---|
| 2.7.x（神农在用） | 2023-11 起无开源安全补丁，裸奔中 |
| 3.5.x | 开源支持止于 2026-06-30 |
| 4.0.x | 2025-11 发布，支持到 2026-12，别停在这 |
| 4.1.x | 2026-06 发布，支持到 2027-07，升级目标定它 |

- **现实路径**：2.7 → 3.5（JDK 17）→ 4.1，别想一步到位；先在 3.5 上把 deprecation warning 清零（4.0 删掉了所有 3.x 标记废弃的 API）。
- **三件硬骨头**：① JDK 8→17（`--release`、反射/封装 API）；② `javax.*` → `jakarta.*` 全量改包名；③ 依赖对齐——Flowable 6.7.2 只能跑 Boot 2.7，上 Boot 3 必须升 Flowable 7.x；MyBatis-Plus / Druid 也要换支持 Boot 3（Jakarta）的版本，升级前逐个核对 release notes（待验证）。
- **收益对得上神农这种 IO 密集型**：Boot 3.2+ 虚拟线程一键开（`spring.threads.virtual.enabled=true`），Tomcat 请求线程换虚拟线程，调 OceanBase/下游接口的等待不再占平台线程——和笔记里"虚拟线程"一节呼应。

> 来源（2026-10-20 核验）：[spring-boot-4](https://www.danvega.dev/blog/spring-boot-4-is-here)；[spring-boot-3.5 EOL](https://dev.to/feezan_khattak/spring-boot-35-stopped-getting-patches-in-june-dont-migrate-to-40-4ba)；[boot-4.0.8](https://spring.io/blog/2026/08/20/spring-boot-4-0-8-available-now/)；[SimpleAsyncTaskExecutor Javadoc](https://docs.spring.io/spring/docs/5.3.24/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html)；[Flowable 7.0.0](https://www.flowable.com/blog/releases/flowable-open-source-7-0-0-release)

## 小技巧与冷知识

【小技巧】`--debug` 只在排障时开：它打出的 Condition Evaluation Report 会列出全部自动配置的生效/未生效原因，启动日志会膨胀几倍。本地排障用 `--debug`，线上想看某次启动的判定结果，用 `actuator/conditions` 端点按需查，别长期开着 debug 跑生产。

【小技巧】`@SpringBootApplication` 是三个注解的组合：`@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`。只想关掉某一个自动配置时，在它上面写 `exclude = DataSourceAutoConfiguration.class`，而不是把整个 starter 删掉——删 starter 会连带丢掉一串传递依赖。

【小技巧】`spring-boot-starter` 本身不带 Web、JDBC 这些功能：它只带核心依赖（spring-boot、spring-boot-autoconfigure、日志），真正的业务能力靠 `spring-boot-starter-web`、`spring-boot-starter-data-jpa` 这类功能 starter 叠上去。项目“能启动但接口 404”时，先看 web starter 在不在依赖里，而不是去查 Controller 写没写错。

【小技巧】`@ConfigurationProperties` 绑定失败时启动就报错，报错信息会指出是哪个前缀、哪个字段配错了；`@Value` 拼错 key 只会注入 null 或默认值，启动照样通过、运行时才炸。复杂配置优先用前者，启动期校验等于把配置错误提前到部署前暴露。

【小技巧】`AopContext.currentProxy()` 要先开 exposeProxy 才拿得到代理：自调用想走代理除了注入自己，也可以用它，但必须在配置类上加 `@EnableAspectJAutoProxy(exposeProxy = true)`，否则拿到的还是 this，事务照样不生效。原文只给了“注入自己 / currentProxy”两个解法，第二个解法的前提开关容易被漏掉。

【小技巧】自调用失效不只事务一个：`@Async`、`@Cacheable`、`@Transactional` 都是靠代理生效的，同一个类里 `this` 直接调，注解全部绕过。想让异步、缓存、事务在类内调用时也生效，最稳的办法是把被调方法抽到另一个 Bean 里，而不是逐个注解找变通写法。

【小技巧】`application.properties` 优先级高于 yml 只在“同目录、同名”时成立：不同位置（classpath 根目录、`classpath:/config/`、当前目录）也有加载顺序，改了 yml 不生效时除了查同名 properties，也要看是不是别的目录里有同名文件在盖它。

【小技巧】Actuator 的 `loggers` 端点改日志级别是运行期临时生效、重启后恢复：线上临时排障用它很方便，但别把它当成改配置的办法——想持久化改级别还是要改 yml 或配置中心，否则下次重启级别又回去了。

【小技巧】优雅停机 `server.shutdown=graceful` 只对内嵌 Tomcat/Jetty 这类 Web 服务器生效；`spring.lifecycle.timeout-per-shutdown-phase` 默认 30 秒，超时后 Spring 直接关容器，在途请求和异步任务会被打断。停机等待时间按最长任务的耗时来估，别用默认值硬扛。

【小技巧】`@SpringBootTest` 默认启动整个应用上下文，单测一层时太慢：只测 Controller 用 `@WebMvcTest`，只测持久层用 `@DataJpaTest`，它们只加载相关切片，启动速度和全量测试差一个量级。写单元测试先选切片注解，实在要全量上下文再用 `@SpringBootTest`。

【冷知识】Spring Boot 1.0 在 2014 年 4 月发布，核心作者之一是 Phil Webb；它的设计吸收了 Spring Roo、Grails 的“约定优于配置”思路，把“配一堆 XML 才能跑起来”变成了“加一个 starter 就能跑”。名字里的 Boot 来自 bootstrap（引导启动），意思是帮你把应用“引导”起来。

【冷知识】Boot 2.7 是最后一个支持 JDK 8 的主线，Boot 3.x 起步就是 JDK 17；`javax.*` 改 `jakarta.*` 是 Java EE 从 Oracle 转到 Eclipse 基金会、改名 Jakarta EE 后的连锁反应。升级时改包名不是 Spring 一家的事，MyBatis-Plus、Druid、Flowable 这些依赖也要同步换支持 Jakarta 的版本，原文第 5 节的三件硬骨头里第三件就是这个。

## 面试与自测

1. 自己定义的 Bean 为什么会让对应的自动配置退让？自动配置没按预期生效时，本篇推荐用什么手段排查？
2. `@Transactional` 失效的 4 个经典场景各是什么？每个场景的解法是什么？
3. 神农当前是 Boot 2.7.3，本篇给出的现实升级路径是什么？升级路上有哪三件硬骨头？

### 参考答案

1. 因为自动配置带 `@ConditionalOnMissingBean`，自己定义的 Bean 会让对应自动配置退让，这是“明明配了却没生效 / 没配却生效了”类问题的主线。排查手段是启动加 `--debug` 看 Condition Evaluation Report（哪些自动配置生效 / 没生效及原因），或看 `actuator/conditions` 端点。
2. ① 同类自调用：`this` 直接调绕过代理，解法是注入自己 / `AopContext.currentProxy()`；② private / final 方法：代理不了，方法不是 public 同理（CGLIB 代理要求 public）；③ 异常被吞：try-catch 吃掉异常且没抛，事务以为成功，想回滚就 `throw new RuntimeException` 或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`；④ checked 异常：默认只回滚 RuntimeException，受检异常要回滚加 `rollbackFor = Exception.class`。另注意：类没被 Spring 管理（自己 new 出来的对象）时注解全都不生效。
3. 现实路径是 2.7 → 3.5（JDK 17）→ 4.1，别想一步到位；先在 3.5 上把 deprecation warning 清零（4.0 删掉了所有 3.x 标记废弃的 API）。三件硬骨头：① JDK 8→17（`--release`、反射 / 封装 API）；② `javax.*` → `jakarta.*` 全量改包名；③ 依赖对齐——Flowable 6.7.2 只能跑 Boot 2.7，上 Boot 3 必须升 Flowable 7.x，MyBatis-Plus / Druid 也要换支持 Boot 3（Jakarta）的版本，升级前逐个核对 release notes。

## 关联知识

- [Druid 监控与慢 SQL 定位](开发语言/Java/Druid.md)
- [多数据源（dynamic-datasource）](开发语言/Java/多数据源.md)
- [MyBatis-Plus](开发语言/Java/MyBatis/MyBatis-Plus.md)
- [@Async 异步](开发语言/Java/Async-异步.md)
- [虚拟线程](开发语言/Java/虚拟线程.md)
