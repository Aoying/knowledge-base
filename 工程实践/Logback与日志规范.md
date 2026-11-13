---
知识点: Logback 与日志规范
层级: 进阶
适用版本: Logback 1.2 / Spring Boot 2.7.3
更新日期: 2026-11-14
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
- 日志是可观测性三支柱之一：指标发现问题、链路缩小范围、**日志定位根因**——规范的全部目的就是让这一步快，见 [监控告警](工程实践/运维/监控告警.md)。

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

- **绝不入日志**：密码、token/密钥、完整银行卡号、完整身份证号、手机号全号、短信验证码。框架日志也要防：MyBatis/Druid 的 SQL 日志会把参数原样打出，生产一律关闭 SQL 明文日志（与 [Spring Boot 篇](开发语言/Java/Spring/Spring-Boot.md)「SQL log 只在本地开」同一条纪律）。
- 需要打的标识字段先脱敏：手机号 `138****8000`、身份证前 3 后 4、卡号只留后 4。写统一 `SensitiveMask` 工具收口，禁止各业务点手写截取（手写必漏）。
- 异常信息与接口回参同样过脱敏；确需留存敏感上下文的走带权限的审计日志通道，与运行日志分开存放、分开授权。日志文件本身也按敏感数据对待（访问授权、保留期、归档加密）。

## 6. 疑难杂症

- **日志打了两遍**：某个 logger 既配了自己的 Appender 又继承 root 的 Appender 且 `additivity` 默认 true，输出两次。给该 logger 设 `additivity="false"`。
- **改了 logback-spring.xml 不生效**：查是否用 `logging.config` 指到了别的文件；查 classpath 根目录是否还有 `logback.xml` 抢先被原生 Logback 加载。
- **类路径绑定冲突**：出现 `multiple bindings` 或 logback 类的 `ClassCastException`，多半是 log4j-slf4j-impl 与 logback-classic 共存。用 `mvn dependency:tree | grep -i -E "log4j|slf4j|logback"` 拉清单，Boot 项目统一由 starter-logging 收口，其他门面只留桥接到 slf4j 的包。
- **生产临时要开 DEBUG**：用 Actuator 的 `loggers` 端点运行时改级别、事后复位，前提是端点做了访问控制（见 Spring Boot 篇）。
- **容器里日志去哪**：打到 stdout 由平台统一采集，或落文件由采集 agent 搬走，二选一并全环境统一，混用会有一部分日志查不到。见 [Docker](工程实践/运维/Docker.md)。

## 7. 常见误区

- 误区：日志越多越好查。日志量与磁盘、IO、采集成本成正比，无节制的 INFO 会淹没真正的错误行；「关键节点打、细节靠 DEBUG 动态开」才是可持续的策略。
- 误区：异步 Appender 包上就万事大吉。丢弃阈值、队列深度、停机丢弃三个行为都得按日志重要性单独评估，账务审计类日志不适合与普通运行日志共用一条异步通道。
- 误区：traceId 只在本服务打就够了。不向下透传，跨服务排查照样断链；不进 MDC 只拼在个别日志行里，框架与第三方日志行照样没有，串不起来。

## 8. 日志体量定量：滚动、队列与磁盘量算一算

本节只做定量换算，不重复第 2 节的配置写法与第 3 节的级别口径。换算口径：容量按十进制近似，1GB = 1000MB、1MB = 1000KB；Logback 的 FileSize 实际按 1024 进制，同样一组数按 1024 算约大 2.4%，不影响量级判断。行体量、速率、占比均为估算口径，生产值以实测为准。

### 8.1 滚动体量：100MB、30 天、20GB，谁先到

第 2 节的三个数各管一头：`maxFileSize 100MB` 管单个文件多大就切，`maxHistory 30` 管最多保留 30 天，`totalSizeCap 20GB` 管归档总量封顶。30 天是上限，不是保证——每天写得多，封顶会先到。

- 封顶对应的文件数：20GB ÷ 100MB = 200 个文件（按 1024 进制约 205 个）。
- 30 天刚好不触顶的日写量：20GB ÷ 30 天 ≈ 0.67GB/天（按 1024 进制约 0.68GB/天）。
- 每天写 0.5GB：封顶够 40 天，`maxHistory` 先到，保留 30 天。
- 每天写 1GB：封顶够 20 天，保留约 20 天。
- 每天写 2GB：封顶够 10 天，保留约 10 天。
- 每天写 5GB：封顶够 4 天，保留约 4 天。
- 每天写多少、每天切几个文件：日写量 ÷ 100MB；写 1GB/天约切 10 个文件，写 2GB/天约切 20 个文件。

【算例 1】某服务每天写 2GB 日志。封顶保留天数 = 20GB ÷ 2GB/天 = 10 天；`maxHistory` 的 30 天够不着，先触顶的是 `totalSizeCap`，实际保留约 10 天，第 11 天前的日志已被删。想保住 30 天，日写量须压到约 0.67GB/天以下。

结论就一句：保留天数 = min（30 天，20GB ÷ 日写量），日写量是唯一变量。

### 8.2 AsyncAppender 队列：256 条能顶多久

官方默认值与第 2 节参考配置是两套数，别混用：默认 `queueSize` 256、`discardingThreshold` 20%；参考配置显式写了 `queueSize` 1024、`discardingThreshold` 0。

- 默认丢弃触发点：队列剩余容量小于 20% 时丢 TRACE/DEBUG/INFO，只留 WARN/ERROR。256 × 20% = 51 条剩余，对应已占约 205 条（256 − 51 = 205）。
- 参考配置没有丢弃触发点：阈值 0 意味着一条不丢，但队列满了会堵业务线程（`neverBlock` 默认 false），见第 2 节要点。
- 队列 1024 若仍用默认 20% 阈值，触发点是已占约 819 条（1024 × 80%），剩余约 205 条——这是“把队列翻 4 倍，丢弃触发点也翻 4 倍”的换算，不是参考配置的行为。
- 队列的单位是条数，不是字节数：一条带堆栈的大日志与一条短日志各占 1 个槽位。

【算例 2】生产 1000 条/秒、落盘消费 500 条/秒，净增 500 条/秒。默认队列 256 条约 0.5 秒打满（256 ÷ 500 ≈ 0.51 秒），且在约 205 条、即约 0.4 秒时已开始丢 TRACE/DEBUG/INFO；参考配置队列 1024 条约 2.0 秒打满（1024 ÷ 500 ≈ 2.05 秒），不丢弃但满了就堵业务线程。排空时间同理：满队列 256 条按 500 条/秒消费约 0.5 秒排空，1024 条约 2.0 秒。

队列只能扛住零点几秒到几秒的突发，扛不住持续超速：生产长期高于消费，任何队列都会满，区别只是丢日志还是堵线程。

### 8.3 日志行体量：每行多少字节 × 多少条/秒

公式只有一行：日写量 = 每行字节数 × 条/秒 × 86400 秒。行体量取 200 字节/行是估算口径：原文 pattern 带时间戳、线程名、级别、traceId 与类名，一行约 150–250 字节，取 200 字节作中值；带堆栈的行按数 KB 另算。

- 10 条/秒 × 200 字节 ≈ 0.17GB/天，30 天约 5.2GB。
- 50 条/秒 × 200 字节 ≈ 0.86GB/天，30 天约 25.9GB。
- 100 条/秒 × 200 字节 ≈ 1.73GB/天，30 天约 51.8GB。
- 500 条/秒 × 200 字节 ≈ 8.64GB/天，30 天约 259GB。
- 同一条/秒下行体量翻倍，日写量跟着翻倍：100 条/秒、500 字节/行约 4.32GB/天；1000 字节/行约 8.64GB/天。

【算例 3】100 条/秒、每行 200 字节：日写量 = 200 × 100 × 86400 = 1,728,000,000 字节 ≈ 1.73GB/天；30 天累计约 51.8GB，已超 20GB 封顶，实际保留约 11.6 天（20 ÷ 1.73 ≈ 11.6）。这组数与 8.1 节的“每天写 2GB 只保留 10 天”在同一量级，互相印证。

### 8.4 采样与级别过滤：关掉 DEBUG 能省多少

关 DEBUG 省多少，没有通用实测比例，只能按“DEBUG 占总条数几成”这个条件换算：省下的比例就是 DEBUG 的占比，剩下的比例是 1 减去该占比。占比必须按本服务实测，下表只是条件换算，不是基准测试数值。

- DEBUG 占 30%：关掉后剩 70%，1.73GB/天降到约 1.21GB/天。
- DEBUG 占 50%：关掉后剩 50%，1.73GB/天降到约 0.86GB/天。
- DEBUG 占 70%：关掉后剩 30%，1.73GB/天降到约 0.52GB/天。
- DEBUG 占 90%：关掉后剩 10%，1.73GB/天降到约 0.17GB/天。
- 采样是同一种换算：对某类日志采样 10%，就是把该类条数乘以 0.1；1000 条/秒采 10% 后剩 100 条/秒，200 字节/行时约 1.73GB/天。

【算例 4】某服务 100 条/秒，其中 DEBUG 占 70%（假设口径，以实测为准）。关掉 DEBUG 后剩 30 条/秒，日写量 = 200 × 30 × 86400 = 518,400,000 字节 ≈ 0.52GB/天，比原来 1.73GB/天省约 70%；按 8.1 节换算，保留天数从约 11.6 天回到 30 天（0.52GB/天 × 30 天 ≈ 15.6GB，未超 20GB 封顶）。

级别过滤省的是条数，不是单行字节数；行里塞大对象、打全量入参，关级别也省不掉单行变大的那部分，见第 3 节写法纪律。

- 边界条件 1：官方文档明确，`maxHistory` 的限制先执行、`totalSizeCap` 的限制后执行；封顶再宽也突破不了 30 天，30 天是硬上限。
- 边界条件 2：`totalSizeCap` 只计归档文件总量，不计当前正在写的活动文件；磁盘占用峰值还要再加一个未归档文件，最多约 100MB。
- 边界条件 3：丢弃阈值只丢 TRACE/DEBUG/INFO，WARN/ERROR 不会被阈值丢弃；但队列彻底满且 `neverBlock` 为 false 时，所有级别都会堵业务线程，见 8.2 节。
- 边界条件 4：本节每行 200 字节、DEBUG 占比、生产/消费速率均为估算口径；带堆栈的行可达数 KB，压缩与采集会改变实际落盘量，生产值以实测为准，不要把估算数当作基准测试数值。

## 小技巧与冷知识

【小技巧】占位符只省了拼接，参数表达式照样求值：`log.debug("结果 {}", toJson(bigObj))` 即使 DEBUG 关着，toJson 也会先跑完再决定不打印。原文只讲了占位符是惰性的，没提参数求值是急切的——拼大对象、调序列化这类贵操作，先用 `if (log.isDebugEnabled())` 包一层再打，级别关闭时连求值都省了。

【小技巧】AsyncAppender 的 discardingThreshold 设 0 保不丢，但队列满了会堵业务线程：neverBlock 默认是 false，队列塞满时调用 log 的业务线程会阻塞在入队上，等落盘线程腾出位置才继续。原文只讲了丢弃阈值和 kill -9 丢队列，没提这一层——高峰期日志突增时，“异步”等于把落盘压力传回了业务线程；queueSize 要按峰值留够，或接受 neverBlock=true 丢日志的 trade-off，两者只能二选一。

【小技巧】改配置不用重启，用 scan 热加载：`<configuration scan="true" scanPeriod="30 seconds">`，改级别、改 appender 后等一轮扫描周期即生效。生产上临时要开 DEBUG、又不方便走 Actuator 端点时，改文件等半分钟是最省事的办法；scan 默认周期是 1 分钟，scanPeriod 可以按需调小。

【小技巧】想把某一级别单独分流，用 LevelFilter 别用 ThresholdFilter：ThresholdFilter 是“大于等于”，配 WARN 会把 ERROR 也一起收进来；LevelFilter 精确匹配 level，onMatch=ACCEPT、onMismatch=DENY，只收 WARN 这一档。单级别分文件（比如 WARN 单独一份告警复核日志）只能靠 LevelFilter，ThresholdFilter 做不到。

【小技巧】配置里的变量写上默认值，本地没配也不启动失败：`${LOG_LEVEL:-INFO}`、`${LOG_DIR:-logs}` 这种 `:-` 写法，没配环境变量时自动用默认值兜底。同一份 logback-spring.xml 跑遍本地、测试、生产，差异只靠环境变量，少了“这台机器忘配变量、启动就报错”的翻车。

【冷知识】SLF4J 的 `{}` 占位符有个“最后参数是异常就当堆栈”的规则：`log.error("审批失败 applyId={}", applyId, e)` 里只有一个占位符、两个参数，e 不会被当成第二个占位符的值，而是按异常打堆栈；占位符个数等于参数个数时，最后一个参数才当普通值。写错的表现是日志里少一个字段、多一段堆栈，排障先数占位符个数。

【冷知识】配置改了不生效，先开 debug 看 Logback 内部状态：`<configuration debug="true">` 会在控制台打印它加载了哪个文件、哪个 appender 初始化失败、哪行配置解析报错。原文只讲了查 logging.config 和 logback.xml 抢加载，debug 输出是更直接的证据——配置文件到底有没有被读到、读到哪一步卡住，一眼就能确认。

【冷知识】SLF4J 2.0 的流式 API 在本篇的版本下不可用：`log.atInfo().addKeyValue("loanId", id)` 这类写法要 SLF4J 2.0 + Logback 1.3 才支持，Spring Boot 2.7.3 用的是 SLF4J 1.7 + Logback 1.2，照抄网上的例子会编译不过。结构化键值对、流式日志 API 是升级 Boot 3.x 之后才有的东西，2.7 的项目别按新版文档写代码。

## 实战案例

### 案例一：高峰期一段 INFO 日志凭空消失，是队列丢了还是滚动删了

以下为设计演练示例，非真实生产事故。

**现象**

高峰过后复盘，发现某时段的 INFO 日志成段缺失，对不上业务流水；同一时段的 WARN、ERROR 日志还在，服务本身没有重启、没有报错。

**排查**

1. 先分清是“丢了”还是“被删了”：按 §8.1 的换算，保留天数 = min（30 天，20GB ÷ 日写量）。每天写 2GB 日志时，20GB 封顶只够 10 天，超过的旧日志是滚动策略按 `totalSizeCap` 删掉的，不是队列丢的；缺失的是最近几小时的日志，才往队列方向查。
2. 对照配置用的是哪一套数：默认 AsyncAppender 是 `queueSize` 256、`discardingThreshold` 默认 20%；§2 参考配置显式写了 `queueSize` 1024、`discardingThreshold` 0，两套行为完全不同，别混用（§8.2）。
3. 按 §8.2 的算例口径换算：生产 1000 条/秒、落盘消费 500 条/秒，净增 500 条/秒。默认队列约 0.5 秒打满，而且在约 205 条、即约 0.4 秒时就已开始丢 TRACE/DEBUG/INFO；参考配置队列 1024 条约 2.0 秒打满。

**根因**

默认丢弃阈值的行为：队列剩余空间小于 20% 时丢弃 TRACE/DEBUG/INFO，只保留 WARN/ERROR（§2 要点）。INFO 成段缺失、WARN/ERROR 还在，正是这个行为的特征。队列只能扛住零点几秒到几秒的突发，生产长期高于消费时，任何队列都会满，区别只是丢日志还是堵线程（§8.2）。

**修复**

- 审计类、账务类日志不能容忍丢弃：显式设 `discardingThreshold` 为 0，或单独走同步 Appender（§2 要点）。
- 普通运行日志若接受丢弃，保留默认阈值也可以，但必须是知情的选择，不能以为“异步包上就万事大吉”（§7）。
- 治本要降日写量：按 §8.4 关掉生产 DEBUG 或做采样。§8.4 算例 4 的口径：100 条/秒、DEBUG 占 70% 时，关掉 DEBUG 后日写量从约 1.73GB/天降到约 0.52GB/天，30 天约 15.6GB，未超 20GB 封顶，保留天数回到 30 天。
- `discardingThreshold` 设 0 保不丢，但队列满了会堵业务线程（`neverBlock` 默认 false）；`queueSize` 要按峰值留够，或接受 `neverBlock=true` 丢日志的 trade-off，两者只能二选一。

**复盘**

- 排查顺序：先算保留天数分清“丢”与“删”，再对配置、再算队列打满时间；三步的数字全部来自 §8，缺哪一步都容易修错地方。
- 队列的单位是条数不是字节数，一条带堆栈的大日志与一条短日志各占 1 个槽位（§8.2）；只看磁盘占用判断不了队列问题。
- 本案例的速率与日写量均为 §8 已核验的算例口径，不是真实压测数据；生产值以实测为准。

### 案例二：grep 一个 traceId 只拼出半条链路，还撞见一行手机号全号

以下为设计演练示例，非真实生产事故。

**现象**

按一个 traceId 全量 grep，只能拼出入口处的几行日志，下游调用与异步分支的日志里 traceId 为空；复核同一批日志时，还发现某一行里出现了手机号全号。

**排查**

1. 先看格式：§2 参考配置的 pattern 是否带 `%X{traceId}`，没有它，MDC 里有值也不会打出来。
2. 再看入口：§4 的 TraceFilter 是否在入口 `MDC.put("traceId", …)`、请求结束是否 `MDC.clear()`；不清会串到同线程的下个请求，是线程池复用的经典污染。
3. 再看异步分支：`@Async`、自建线程池、并行流是否按 §4 配了 `TaskDecorator`，提交时复制 MDC、执行前装上、执行后清理；漏了的表现正是异步日志 traceId 为空。
4. 再看跨服务：调用下游时是否把 traceId 放进 HTTP 头，下游过滤器是否优先取头里的值；不向下透传，跨服务排查照样断链（§7）。
5. 敏感字段另查一条线：是否 MyBatis/Druid 的 SQL 日志把参数原样打了出来，生产是否关闭了 SQL 明文日志（§5）；业务代码是否绕过统一的 `SensitiveMask` 工具手写截取（§5，手写必漏）。

**根因**

- 链路断在两处：MDC 底层是 ThreadLocal 的键值表，值在父线程里，子线程拿不到；跨服务不透传请求头，下游就生成了新的 traceId。两处各断一次，grep 只能拼出半条链路。
- 手机号全号进日志，是脱敏收口漏了：SQL 明文日志或手写截取绕过了统一脱敏，单点修补拦不住下一处。

**修复**

- 线程池配 `TaskDecorator`，提交时复制 MDC、执行前装上、执行后清理（§4）；跨服务透传 traceId，全链路共用一个 traceId（§4）。
- 生产一律关闭 SQL 明文日志，慢 SQL 走 Druid 监控而非全量日志（§6 自测参考答案口径）；需要打的标识字段先脱敏再打，手机号 `138****8000`、身份证前 3 后 4、卡号只留后 4，统一走 `SensitiveMask` 收口（§5）。
- 确需留存敏感上下文的走带权限的审计日志通道，与运行日志分开存放、分开授权（§5）。

**复盘**

- traceId 排查有固定顺序：pattern → 入口 MDC → 异步 TaskDecorator → 跨服务透传；跳过前一步直接改后一步，修完还是断的。
- 脱敏是收口问题：只堵住被发现的那一行，下一个手写截取或 SQL 日志还会漏；必须回到统一工具 + 生产关 SQL 明文日志这两条纪律上。
- `TaskDecorator` 的具体实现原文未展开，仅作概念提示；Boot 2.7 用 Spring Cloud Sleuth 时 traceId 由它自动生成并注入 MDC，与手写过滤器二选一，别叠两套（§4）。

## 9. 面试与自测

- `logback.xml` 与 `logback-spring.xml` 的区别？为什么 Boot 项目用后者？
- AsyncAppender 什么时候会丢日志？哪类日志不能走它？
- MDC 的原理？线程池里为什么会丢 traceId、怎么补？
- 日志重复输出的成因与修法？
- 生产环境 SQL 日志为什么必须关？与排障需求怎么平衡（慢 SQL 走 Druid 监控而非全量日志）？

- 按第 2 节配置（单文件 100MB、保留 30 天、总量封顶 20GB），某服务每天写 2GB 日志，实际能保留几天、为什么不是 30 天？若改用默认 AsyncAppender（队列 256、丢弃阈值 20%），队列占到多少条时开始丢 TRACE/DEBUG/INFO，生产 1000 条/秒、消费 500 条/秒时多久打满？

### 参考答案

1. logback.xml 由原生 Logback 直接加载，不认 Boot 的扩展；logback-spring.xml 支持 Boot 扩展的 springProperty、springProfile，且加载时机在环境准备好之后。所以 Boot 项目用后者，才能按 profile 切配置、用配置项给日志参数。
2. AsyncAppender 有反直觉默认：队列剩余空间小于 20% 时丢弃 TRACE/DEBUG/INFO，只保留 WARN/ERROR；discardingThreshold 设为 0 则一条不丢。另外进程被 kill -9 时队列里未落盘的日志会丢，停机要走正常关闭刷队列。审计类、账务类日志不能容忍丢弃，应显式设 0，或单独走同步 Appender。
3. MDC 底层是 ThreadLocal 的键值表，pattern 里用 %X{traceId} 就能每行自动带上。线程池里会丢 traceId，因为 MDC 在父线程，子线程拿不到；而且不清还会串到同线程的下个请求（线程池复用的经典污染）。补法：入口过滤器生成 traceId 放入 MDC、请求结束 MDC.clear()；给线程池配 TaskDecorator，提交时复制 MDC、执行前装上、执行后清理。
4. 成因是某个 logger 既配了自己的 Appender，又继承 root 的 Appender，且 additivity 默认 true，导致输出两次。修法：给该 logger 设 additivity="false"。如果改了配置不生效，再查 logging.config 是否指到了别的文件，以及 classpath 根目录是否还有 logback.xml 抢先被原生加载。
5. MyBatis/Druid 的 SQL 日志会把参数原样打出，密码、卡号、身份证号等敏感信息会直接进日志，生产必须一律关闭 SQL 明文日志（SQL log 只在本地开）。排障需求不靠全量 SQL 日志平衡：慢 SQL 走 Druid 监控，确需留存敏感上下文的走带权限的审计日志通道，与运行日志分开存放、分开授权。

6. 每天 2GB 时 20GB 封顶只够 10 天，先触顶的是 totalSizeCap，30 天的 maxHistory 够不着。默认队列 256 条，剩余 20% 即 51 条时开始丢弃，对应已占约 205 条，只丢 TRACE/DEBUG/INFO。净增 500 条/秒时约 0.5 秒打满。

## 10. 关联知识

### 前置知识

读本篇前建议先有以下基础，括号内是本篇用到它的那一点；路径均已在仓库 tree.json 中核对存在。

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)：Boot 默认日志体系、`logback-spring.xml` 的 `<springProperty>`/`<springProfile>` 扩展、Actuator 的 `loggers` 端点动态调级、SQL 日志只在本地开的纪律（§1、§2、§6）。
- [Java](开发语言/Java/Java.md)：Logger 按类名分层、占位符写法、异常作最后一个参数打堆栈等 Java 基础写法（§1、§3）。
- [ThreadLocal](开发语言/Java/并发/ThreadLocal.md)：MDC 底层就是 ThreadLocal 的键值表，理解它才能理解 traceId 为什么会串到同线程的下个请求（§4）。
- [线程池](开发语言/Java/并发/线程池.md)：`@Async` 与自建线程池的提交/执行模型，是理解 MDC 在异步场景丢失、以及 `TaskDecorator` 复制上下文的前提（§4）。
- [Java 异常体系](开发语言/Java/Java-异常体系.md)：异常必须带堆栈、只打 `e.getMessage()` 等于扔掉定位信息，这条纪律的异常侧依据（§3）。
- [Docker](工程实践/运维/Docker.md)：容器里日志打 stdout 由平台采集、还是落文件由采集 agent 搬走，二选一并全环境统一（§6）。

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)：默认日志体系、Actuator loggers 端点、SQL 日志的开关纪律。
- [监控告警](工程实践/运维/监控告警.md)：日志在可观测三支柱中的位置与告警分级原则。
- [@Async 异步](开发语言/Java/Async-异步.md)：异步场景的上下文（含 MDC）传递问题。
- [Docker](工程实践/运维/Docker.md)：容器化部署下日志的输出与采集路线。
