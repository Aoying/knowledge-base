---
知识点: Spring Boot
层级: 进阶
适用版本: Spring Boot 2.7.3
更新日期: 2026-10-22
关联:
  - Java
  - MyBatis-Plus
  - Java知识点地图
标签:
  - Java
  - 实践
---

# Spring Boot

## Spring Boot 2.7 实战

### 配置加载顺序（后覆盖前）
1. 命令行参数 `--server.port=8081`
2. `SPRING_APPLICATION_JSON` / 系统环境变量
3. `application-{profile}.yml` → `application.yml`

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

### @Transactional 失效的 4 个经典场景
1. **同类自调用**：`this` 直接调，绕过代理。解法：注入自己 / `AopContext.currentProxy()`。
2. **private / final 方法**：代理不了，不生效。
3. **异常被吞**：try-catch 吃掉异常且没抛，事务以为成功。想回滚就 `throw new RuntimeException` 或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
4. **checked 异常**：默认只回滚 RuntimeException。受检异常要回滚加 `rollbackFor = Exception.class`。

### 事务传播常用组合
- `REQUIRED`（默认）：有就加入，没有新建。
- `REQUIRES_NEW`：挂起当前，永远新建——记流水、审计日志独立提交用它。
- 注意：`REQUIRES_NEW` 内部抛异常不影响外部已提交的事务（已挂起隔离）。

## Spring Boot 2.7 → 升级路线图（2026-10-20 核验）

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

