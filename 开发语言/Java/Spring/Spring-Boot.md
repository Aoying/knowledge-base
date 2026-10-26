---
知识点: Spring Boot
层级: 进阶
适用版本: Spring Boot 2.7.3
更新日期: 2026-10-23
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

## 关联知识

- [Druid 监控与慢 SQL 定位](开发语言/Java/Druid.md)
- [多数据源（dynamic-datasource）](开发语言/Java/多数据源.md)
- [MyBatis-Plus](开发语言/Java/MyBatis/MyBatis-Plus.md)
- [@Async 异步](开发语言/Java/Async-异步.md)
- [虚拟线程](开发语言/Java/虚拟线程.md)
