---
知识点: Logback 与日志规范
层级: 进阶
适用版本: Logback 1.2 / Spring Boot 2.7.3
更新日期: 2026-10-23
关联:
  - Spring-Boot
  - 监控告警
  - Docker
标签:
  - 运维
  - 实践
  - 排错
---

# Logback 与日志规范

## 1. 核心概念

- 门面与实现分开：代码只依赖 **SLF4J** API，运行期实现用 **Logback**（Spring Boot 2.7 默认）；类路径里多个绑定或桥接混乱是日志类故障的头号来源，见第 6 节。
- 四件套：`Logger`（按类名分层、继承级别）、`Appender`（输出到哪）、`Encoder/Layout`（格式）、`Filter`（过滤）。
- 配置文件优先用 `logback-spring.xml` 而非 `logback.xml`：前者支持 Boot 扩展的 `<springProperty>`、`<springProfile>`，且加载时机在环境准备好之后。
- 日志是可观测性三支柱之一：指标发现问题、链路缩小范围、**日志定位根因**——规范的全部目的就是让这一步快，见 [监控告警](工程实践/监控告警.md)。

## 2. 参考配置：滚动 + 异步

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="APP" source="spring.application.name" defaultValue="app"/>
    <property name="LOG_DIR" value="logs"/>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/${APP}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_DIR}/${APP}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>   <!-- 按天 + 按大小双滚动 -->
            <maxHistory>30</maxHistory>         <!-- 保留 30 天 -->
            <totalSizeCap>20GB</totalSizeCap>   <!-- 总量封顶防写满磁盘 -->
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 异步包裹：业务线程只塞队列，落盘由后台线程做 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>1024</queueSize>
        <discardingThreshold>0</discardingThreshold> <!-- 0 = 一条不丢，见下方说明 -->
        <appender-ref ref="FILE"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="ASYNC_FILE"/>
    </root>
</configuration>
```

本地开发想同时打控制台，再加一个 `ConsoleAppender` 并用 `<springProfile name="local">` 包住对应的 root 段切换即可。

要点：

- 滚动策略必须配 `maxHistory` + `totalSizeCap` 双保险，日志写满磁盘把服务拖死是真实事故类型，不是理论风险。
- `AsyncAppender` 有个反直觉默认：队列剩余空间小于 20% 时丢弃 TRACE/DEBUG/INFO（`discardingThreshold` 默认即此行为），保留 WARN/ERROR。审计类、账务类日志不能容忍丢弃时显式设 0，或单独走同步 Appender。
- 异步日志在进程被 `kill -9` 时队列里未落盘的会丢；停机走正常关闭（Logback 有关闭钩子刷队列），别拿强杀当常规停机手段。

## 3. 日志规范：级别、格式、内容

### 3.1 级别用法

| 级别 | 打什么 | 不打什么 |
| --- | --- | --- |
| ERROR | 影响业务结果的失败：交易失败、数据不一致、外部调用彻底失败 | 可重试成功的中间失败（那是 WARN） |
| WARN | 异常但可降级/已兜底：重试后成功、参数非法被拦截、慢调用 | 正常业务分支 |
| INFO | 关键业务节点：交易受理/完成、状态流转、定时任务起止，量要克制 | 循环内逐条打印、入参全量回显 |
| DEBUG | 开发排查细节，**生产默认关**，靠动态调级临时开 | —— |
| TRACE | 框架级细节，基本不用 | —— |

判断标准一条：半夜被告警叫起来看到这条日志，它配不配？ERROR 必须对应「需要人介入」，否则就是在制造告警疲劳（与监控篇的告警原则同一条）。

### 3.2 写法纪律

- **占位符，不拼字符串**：`log.info("受理成功 loanId={}", loanId)`。拼接在级别关闭时也照常执行字符串构造，占位符是惰性的。
- **异常必须带堆栈**：`log.error("审批提交失败 applyId={}", applyId, e)`，异常作最后一个参数。只打 `e.getMessage()` 等于扔掉定位信息；也不要 `e.printStackTrace()`（绕过日志体系、无时间戳无级别）。
- **一条事件一行**，字段用 `key=value` 便于 grep 与采集解析；多行对象 toString 回显禁止进 INFO。
- 入口打受理、出口打结果，中间过程只在 DEBUG；异常日志必须带业务主键（申请号、脱敏后的客户号），没有主键的错误日志等于没打。

## 4. MDC 与 traceId 链路

- MDC（Mapped Diagnostic Context）底层是 ThreadLocal 的键值表，值放进去后 pattern 里用 `%X{traceId}` 就能每行自动带上——这是日志能按链路串起来的机制基础。
- 落法：入口过滤器/拦截器为每个请求生成 traceId 放入 MDC，请求结束 `MDC.clear()`（不清会串到同线程的下个请求，是线程池复用的经典污染）：

```java
public class TraceFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        String traceId = req.getHeader("X-Trace-Id");
        MDC.put("traceId", (traceId == null || traceId.isEmpty())
                ? UUID.randomUUID().toString().replace("-", "") : traceId);
        try {
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();
        }
    }
}
```

- **异步/线程池场景 MDC 不自动跟随**：MDC 在父线程里，子线程拿不到。`@Async`、自建线程池、并行流都要处理：给线程池配 `TaskDecorator`，提交时复制 MDC、执行前装上、执行后清理。漏了的表现是异步日志 traceId 为空，见 [@Async 异步](开发语言/Java/Async-异步.md)。
- 跨服务透传：调用下游时把 traceId 放进 HTTP 头，下游过滤器优先取头里的值——全链路共用一个 traceId，`grep <traceId>` 就能拼出全貌。Boot 2.7 用 Spring Cloud Sleuth 时 traceId/spanId 由它自动生成并注入 MDC 与请求头；手写过滤器与之二选一，别叠两套。

## 5. 脱敏纪律（金融项目红线）

- **绝不入日志**：密码、token/密钥、完整银行卡号、完整身份证号、手机号全号、短信验证码。框架日志也要防：MyBatis/Druid 的 SQL 日志会把参数原样打出，生产一律关闭 SQL 明文日志（与 [Spring Boot 篇](开发语言/Java/Spring-Boot.md)「SQL log 只在本地开」同一条纪律）。
- 需要打的标识字段先脱敏：手机号 `138****8000`、身份证前 3 后 4、卡号只留后 4。写统一 `SensitiveMask` 工具收口，禁止各业务点手写截取（手写必漏）。
- 异常信息与接口回参同样过脱敏；确需留存敏感上下文的走带权限的审计日志通道，与运行日志分开存放、分开授权。日志文件本身也按敏感数据对待（访问授权、保留期、归档加密）。

## 6. 疑难杂症

- **日志打了两遍**：某个 logger 既配了自己的 Appender 又继承 root 的 Appender 且 `additivity` 默认 true，输出两次。给该 logger 设 `additivity="false"`。
- **改了 logback-spring.xml 不生效**：查是否用 `logging.config` 指到了别的文件；查 classpath 根目录是否还有 `logback.xml` 抢先被原生 Logback 加载。
- **类路径绑定冲突**：出现 `multiple bindings` 或 logback 类的 `ClassCastException`，多半是 log4j-slf4j-impl 与 logback-classic 共存。用 `mvn dependency:tree | grep -i -E "log4j|slf4j|logback"` 拉清单，Boot 项目统一由 starter-logging 收口，其他门面只留桥接到 slf4j 的包。
- **生产临时要开 DEBUG**：用 Actuator 的 `loggers` 端点运行时改级别、事后复位，前提是端点做了访问控制（见 Spring Boot 篇）。
- **容器里日志去哪**：打到 stdout 由平台统一采集，或落文件由采集 agent 搬走，二选一并全环境统一，混用会有一部分日志查不到。见 [Docker](工程实践/Docker.md)。

## 7. 常见误区

- 误区：日志越多越好查。日志量与磁盘、IO、采集成本成正比，无节制的 INFO 会淹没真正的错误行；「关键节点打、细节靠 DEBUG 动态开」才是可持续的策略。
- 误区：异步 Appender 包上就万事大吉。丢弃阈值、队列深度、停机丢弃三个行为都得按日志重要性单独评估，账务审计类日志不适合与普通运行日志共用一条异步通道。
- 误区：traceId 只在本服务打就够了。不向下透传，跨服务排查照样断链；不进 MDC 只拼在个别日志行里，框架与第三方日志行照样没有，串不起来。

## 8. 面试与自测

- `logback.xml` 与 `logback-spring.xml` 的区别？为什么 Boot 项目用后者？
- AsyncAppender 什么时候会丢日志？哪类日志不能走它？
- MDC 的原理？线程池里为什么会丢 traceId、怎么补？
- 日志重复输出的成因与修法？
- 生产环境 SQL 日志为什么必须关？与排障需求怎么平衡（慢 SQL 走 Druid 监控而非全量日志）？

### 参考答案

1. logback.xml 由原生 Logback 直接加载，不认 Boot 的扩展；logback-spring.xml 支持 Boot 扩展的 springProperty、springProfile，且加载时机在环境准备好之后。所以 Boot 项目用后者，才能按 profile 切配置、用配置项给日志参数。
2. AsyncAppender 有反直觉默认：队列剩余空间小于 20% 时丢弃 TRACE/DEBUG/INFO，只保留 WARN/ERROR；discardingThreshold 设为 0 则一条不丢。另外进程被 kill -9 时队列里未落盘的日志会丢，停机要走正常关闭刷队列。审计类、账务类日志不能容忍丢弃，应显式设 0，或单独走同步 Appender。
3. MDC 底层是 ThreadLocal 的键值表，pattern 里用 %X{traceId} 就能每行自动带上。线程池里会丢 traceId，因为 MDC 在父线程，子线程拿不到；而且不清还会串到同线程的下个请求（线程池复用的经典污染）。补法：入口过滤器生成 traceId 放入 MDC、请求结束 MDC.clear()；给线程池配 TaskDecorator，提交时复制 MDC、执行前装上、执行后清理。
4. 成因是某个 logger 既配了自己的 Appender，又继承 root 的 Appender，且 additivity 默认 true，导致输出两次。修法：给该 logger 设 additivity="false"。如果改了配置不生效，再查 logging.config 是否指到了别的文件，以及 classpath 根目录是否还有 logback.xml 抢先被原生加载。
5. MyBatis/Druid 的 SQL 日志会把参数原样打出，密码、卡号、身份证号等敏感信息会直接进日志，生产必须一律关闭 SQL 明文日志（SQL log 只在本地开）。排障需求不靠全量 SQL 日志平衡：慢 SQL 走 Druid 监控，确需留存敏感上下文的走带权限的审计日志通道，与运行日志分开存放、分开授权。
## 9. 关联知识

- [Spring Boot](开发语言/Java/Spring-Boot.md)：默认日志体系、Actuator loggers 端点、SQL 日志的开关纪律。
- [监控告警](工程实践/监控告警.md)：日志在可观测三支柱中的位置与告警分级原则。
- [@Async 异步](开发语言/Java/Async-异步.md)：异步场景的上下文（含 MDC）传递问题。
- [Docker](工程实践/Docker.md)：容器化部署下日志的输出与采集路线。
