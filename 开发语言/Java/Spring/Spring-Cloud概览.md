---
知识点: Spring Cloud 概览
层级: 进阶
适用版本: Spring Cloud 2021.0 / Spring Cloud Alibaba 2021.0（Boot 2.7 对应线）
更新日期: 2026-11-12
关联:
  - Spring-Boot
  - 分布式事务
  - 限流熔断与降级
标签:
  - Java
  - 实践
---

# Spring Cloud 概览

## 1. 核心概念

Spring Cloud 不是单个框架，而是一套**分布式系统基础设施的标准件集合**：注册发现、配置管理、网关、声明式调用、负载均衡、限流熔断。它解决的从来不是业务问题，而是「一个系统拆成多个进程之后，进程之间怎么找到对方、怎么统一配置、怎么扛住彼此故障」这类分布式税。

先立结论：组件本身不难，难的是判断**要不要交这份税**。本篇先讲每个组件解决什么，再讲拆分的代价，最后给银行内网单体项目的落点判断。

## 2. 组件全景

| 组件 | 代表实现 | 解决什么问题 |
|---|---|---|
| 注册中心 | Nacos、Eureka | 服务实例动态上下线后，调用方如何知道「现在该调哪台」——实例启动时注册、心跳保活，调用方按服务名查列表，不再写死 IP:端口 |
| 配置中心 | Nacos Config、Apollo | 多实例、多环境的配置统一存放、集中下发；部分配置可动态刷新，改参数不用逐台改文件重启 |
| API 网关 | Spring Cloud Gateway | 统一入口：路由转发、鉴权前置、限流、灰度、跨域、日志埋点收口到一层，后端服务不直面调用方 |
| 声明式调用 | OpenFeign | 把 HTTP 调用写成接口 + 注解，屏蔽拼 URL、序列化、选实例的细节 |
| 负载均衡 | Spring Cloud LoadBalancer | 从注册中心拿到的实例列表里按策略（轮询等）挑一台；是客户端侧均衡，与 Nginx 的服务端代理均衡位置不同 |
| 限流熔断 | Sentinel、Resilience4j | 下游变慢或挂掉时保护自己不被拖垮，见 [限流熔断与降级](开发语言/Java/限流熔断与降级.md) |

- **Nacos 一件顶两件**：同时是注册中心 + 配置中心，这是 Spring Cloud Alibaba 系在国内成主流的原因之一，运维面比 Eureka + Config Server 拼装小。
- **Eureka** 是 Netflix 系的注册中心，纯 AP、结构简单，但在 Spring Cloud 的新版本线里 Netflix 组件整体处于维护状态（待验证：以官方发布说明为准），新项目一般不再选它起步。
- **Gateway 与 Zuul**：Zuul 1.x 是基于 Servlet 的同步阻塞模型，Spring Cloud 后来以响应式（WebFlux）的 Spring Cloud Gateway 取代其位置；Gateway 与 Boot 2.7 的 WebMVC 栈是两套运行时，引入时注意别和现有 Servlet 体系混装冲突。

## 3. 生态现状：Alibaba 系与 Netflix 系

- **Netflix 系**：Eureka、Ribbon、Hystrix、Zuul 这批元老，Ribbon 与 Hystrix 已先后退出 Spring Cloud 发行列车（Hystrix 在 2020.0 线移除，Ribbon 由 LoadBalancer 替代），整体是「存量维护、新项目不推荐起步」。
- **Spring Cloud Alibaba 系**：Nacos（注册 + 配置）、Sentinel（限流熔断）、Seata（分布式事务，见 [分布式事务](数据库/分布式事务.md)）、RocketMQ 绑定等，组件活跃、中文语境资料多，是国内企业项目当前的主流选择。
- 版本对位是高频坑：Spring Cloud、Spring Cloud Alibaba、Spring Boot 三者的版本必须按官方兼容表配套（如 Boot 2.7 对应 Cloud 2021.0 与 Alibaba 2021.0 线），跨线混搭出现的往往是启动期诡异报错。各组件当前最新版本与维护状态标**待验证**，选型时以 Spring 官方与 Alibaba 官方发布页为准，不要照抄旧博客的版本组合。

## 4. 拆分的代价（先算账，再谈组件）

把单体拆成微服务，买到的东西和付出的代价都很具体：

**买到的**：模块可独立部署与扩缩容；团队并行交付时互不阻塞发版；单点故障的影响面可以隔离。

**付出的**（每一条都是长期成本）：
- 一次业务操作跨进程后，本地事务失效，要引入 [分布式事务](数据库/分布式事务.md) 的整套复杂度（或最终一致性改造）；
- 排查问题从一个日志文件变成跨服务链路追踪，日志、监控、告警都要升级一档；
- 部署单元从 1 个变 N 个：CI/CD、配置、注册中心本身都成了要运维的基础设施；
- 网络从进程内方法调用变成不可靠介质：超时、重试、幂等、限流熔断全部从「不用考虑」变成「必须设计」；
- 联调和版本兼容成本：接口一旦多方调用，破坏性变更要走双版本并行。

经验法则：拆分解决的是**组织与规模问题**（团队互相阻塞、单点扩不动），不是代码质量问题。单体内部模块边界混乱，拆成进程后只会变成分布式的混乱，俗称「分布式单体」——两边的缺点全占。

## 5. 落点：银行内网单体什么时候才真需要

以神农 e 贷法人版这类银行内网项目为参照（单体部署、用户量内部可控、强事务、审计与合规要求高），务实判断：

- **默认答案是不拆**。内网系统没有互联网级并发，调用链短、发布窗口统一，单体的事务一致性和排障简单是实打实的优势。把模块边界在单体内先划清（分包、模块间只走接口），是成本最低的「为未来留路」。
- 出现以下信号才认真评估拆分：某一个模块（如影像处理、报表批处理）的资源消耗与主体差异巨大、需要独立扩缩容；多个团队改同一个工程、发版互相排队成为交付瓶颈；某部分有独立的合规/等保隔离要求，必须物理分开。
- 真要拆也只拆一两个边界最清、交互最松的模块，配套一步到位：注册与配置（Nacos）、网关、链路日志。不要一上来按业务域切十几个服务。
- 中间路线值得优先考虑：**模块化单体**——单进程内按域分模块、模块间禁止跨库直查、只通过接口调用。拿到大部分边界收益，不交分布式税；将来真要拆，沿模块边界切开即可。

## 6. 常见误区

- **为了简历/技术时髦而拆**：没有规模与组织问题时上全套微服务，团队会被基础设施运维淹没，业务交付反而变慢。
- **把注册中心当配置中心用**（或反之）：两者职责不同，Nacos 一件顶两件是产品形态，不代表概念上可以混。
- **Feign 调用不设超时与降级**：声明式调用写起来像本地方法，最容易让人忘记它走网络。超时、重试（慎用，非幂等接口不要自动重试）、熔断必须显式配。
- **版本三件套不对位**：Boot、Cloud、Alibaba 跨线混搭，见第 3 节。


## 7. 注册中心心跳与故障摘除定量：租约、探活与摘除窗口算一算

先说结论：三个注册中心的心跳与摘除窗口差了一个数量级——Eureka 默认要 90 秒租约才到期，调用方最坏 3 分钟还在调死实例；Nacos 临时实例 30 秒摘除、15 秒就不再返回给调用方；Consul 没有统一默认值，窗口完全由检查配置决定。选型时先问一句：业务能忍受多长时间的“坏实例流量”，再反推要多密的心跳。

（与原文衔接：原文第 2 节与小技巧节已定性讲过 Eureka 拉取、自我保护与 Nacos 临时/持久实例之分，本节只补数字与算账，不重复定性内容。）

### Eureka：租约 30 秒续一次，90 秒不到就摘

- 客户端每 30 秒发一次心跳续租：`eureka.instance.lease-renewal-interval-in-seconds` 默认 30；租约到期时间 `eureka.instance.lease-expiration-duration-in-seconds` 默认 90，即连续 3 次心跳没收到才到期。（已验证）
- 服务端每 60 秒扫一次过期租约（`eureka.server.eviction-interval-timer-in-ms` 默认 60000），扫到过期租约才真正从注册表删除。（已验证）
- 客户端每 30 秒拉一次注册表（`eureka.client.registry-fetch-interval-seconds` 默认 30），原文小技巧节已讲过。（已验证）

算一笔账（N 为注册实例数，数字已用 python3 复算）：

- 心跳量：每实例每分钟 60/30 = 2 次，N 个实例每分钟 2N 次。N=100 时 200 次/分钟；N=1000 时 2000 次/分钟。
- 摘除窗口：实例在某次心跳后立刻挂掉，最后一次心跳记为 t=0，租约在 t=90 秒到期；驱逐扫描每 60 秒一次，最坏要等到 t=150 秒才从服务端注册表删除。最好情况是租约刚好在扫描时刻到期，即 90 秒。
- 客户端感知：摘除后调用方还要等下一次拉表，最坏再加 30 秒，即 t=180 秒还在按旧表调死实例；若再计入服务端只读缓存刷新（默认 30 秒，待验证），上限约 210 秒。（推断）

自我保护阈值 0.85 的含义：

- 服务端每分钟期望收到的续租数 = 2N，自我保护阈值 = 2N × 0.85 = 1.7N。阈值默认每 15 分钟按当前注册实例数重算一次（`eureka.server.renewal-threshold-update-interval-ms` 默认 15 分钟）。（已验证）
- 一旦上一分钟实际收到的续租数低于阈值，服务端进入自我保护：停止一切摘除，注册表冻结。（已验证）
- 举例：N=100，期望 200 次/分钟，阈值 170 次/分钟。若同时挂掉 16 个实例，实际续租 168 次/分钟 < 170，自我保护触发，剩下 84 个活实例的心跳也保不住那 16 个死实例被摘除——它们会一直留在表里，直到网络恢复或人工干预。只挂 15 个时实际续租恰好 170 次，不触发（触发条件是“低于”阈值）。
- 后果：自我保护开启时，摘除停滞等于把“宁可错杀”换成“宁可放过”：网络分区时活实例不会被误删，但死实例的流量会一直被调用方拿到，直到分区恢复。原文小技巧节“实例明明下线了还在列表里”说的就是这个状态，先看面板提示、别急着重启注册中心。

### Nacos：临时实例 5 秒心跳，15 秒标不健康、30 秒摘除

以下数字针对 1.x 的 HTTP 心跳模型与临时实例（默认类型）；2.x 模型变化见边界条件第 5 条。

- 临时实例客户端每 5 秒发一次心跳（`heart-beat-interval` 默认 5000ms）；15 秒收不到心跳（`heart-beat-timeout` 默认 15000ms），服务端把实例标为不健康；30 秒收不到（`ip-delete-timeout` 默认 30000ms），从服务列表删除。（已验证，源码常量）
- 不健康不等于已摘除：标为不健康后，实例不再返回给调用方（Nacos 官网概念页原话：不健康实例在解析服务时不返回给客户端），但仍在列表里，直到 30 秒才删除。（已验证）
- 持久实例不走这套：不靠心跳，服务端反向探活（TCP/HTTP/MySQL 探针；探活周期以服务端配置为准，本次未逐项核实），探活异常只标不健康、永不自动删除。原文小技巧节已讲过临时与持久之分。（已验证）

算一笔账（python3 复算）：

- 心跳量：每实例每分钟 60/5 = 12 次。1000 个临时实例每分钟 12000 次心跳，约 200 次/秒；每小时 720000 次，每天 17280000 次。同等 1000 实例下是 Eureka 的 6 倍（2000 次/分钟）。
- 摘除窗口：实例挂掉后最坏 30 秒摘除；调用方在 15 秒标不健康时就已拿不到它（订阅推送近实时），所以调用方感知延迟在 15~30 秒量级，比 Eureka 的 90~180 秒快一个数量级。
- 代价就是心跳量：5 秒心跳换 15 秒感知，6 倍于 Eureka 的心跳包量；实例数上千后，注册中心的心跳处理本身就成了要评估的负载。

### Consul：没有统一默认值，窗口由检查配置决定

- Consul 不像 Eureka/Nacos 那样给一套全局心跳默认值：健康检查分两种，TTL 检查是被动等服务自己上报（`ttl` 到期未上报即 critical），HTTP/TCP/gRPC/脚本检查是 agent 按 `interval` 主动探活；`interval` 是必填项、没有默认值，`timeout` 有默认值（HTTP/TCP/gRPC 为 10 秒，脚本为 30 秒）。（已验证）
- `deregister-critical-service-after`：检查持续 critical 满这个时长后，服务连同检查一起从目录注销。关键点：它没有默认值，不配就是空字符串，critical 的服务会一直留在目录里、永不自动注销；配了之后最小值 1 分钟，且负责回收的 reaper 每 30 秒才跑一次，实际注销可能比配置值再晚最多 30 秒。（已验证）
- 举例（推断）：HTTP 检查 `interval = "10s"`、`deregister-critical-service-after = "1m"` 时，实例挂掉后最坏约 10 秒（首次探活失败，含超时）+ 60 秒（critical 满 1 分钟）+ 30 秒（reaper 周期）= 约 100 秒才从目录注销。但调用方若用 `?passing` 查询，检查一变 critical 实例就不再返回，感知延迟只有首次探活失败的时间（约 10 秒量级）。
- Spring Cloud Consul 的对应配置是 `spring.cloud.consul.discovery.health-check-critical-timeout`，文档示例为 30m（即 critical 满 30 分钟才注销），走 Spring Cloud Consul 时以该配置为准。（已验证）

### 三者对照表

| | Eureka | Nacos（临时实例，1.x） | Consul |
|---|---|---|---|
| 心跳/探活间隔 | 客户端心跳 30s | 客户端心跳 5s | 无默认值；HTTP/TCP 检查 `interval` 必填，示例 10s |
| 判定不健康 | 租约 90s 到期才算过期（无单独“不健康”状态） | 15s 无心跳标不健康，不再返回给调用方 | 检查失败即 critical（HTTP 超时默认 10s） |
| 摘除 | 90s 到期 + 驱逐扫描每 60s，最坏 150s；自我保护开启时不摘 | 30s 无心跳删除 | `deregister-critical-service-after` 未配则永不自动摘；配后最小 1m，reaper 每 30s，再加最多 30s |
| 客户端感知延迟 | 最坏约 180s（已验证部分）；含只读缓存上限约 210s（推断） | 15~30s（推送近实时） | `?passing` 查询下约首次探活失败时间；目录注销则按上行 |

选型取舍一句话：与原文第 5 节结论一致——默认不拆、真要拆也只拆边界最清的模块，而一旦拆了，注册中心的心跳与摘除窗口就是调用方拿到坏实例的时间：Eureka 靠拉表天生慢半拍、Nacos 靠推送与密心跳换快感知、Consul 把窗口完全交给检查配置，窗口越短，心跳量与误判越多，没有免费的快。

### 边界条件

1. **心跳间隔与网络抖动**：间隔越小、误判越多。Eureka 90 秒租约容忍连续 2 次心跳丢失（30s×3），Nacos 15 秒标不健康容忍连续 2 次丢失（5s×3）；抖动超过 2 个心跳周期就会误判。调小间隔前先看网络抖动的 p99，不要只看均值。
2. **自我保护与网络分区**：Eureka 自我保护是“宁可放过”——分区时死实例一直留在表里，调用方持续拿到坏地址；关掉自我保护则反过来，分区时活实例可能被误删。生产上默认开启（`enable-self-preservation` 默认 true），关之前先确认分区恢复后有人盯着注册表。
3. **临时实例与持久实例区别**：Nacos 的 5/15/30 只针对临时实例；持久实例不靠心跳、不断线不摘除，只标不健康。把 Dubbo 长连接服务注册成临时实例、或把普通 HTTP 服务注册成持久实例，摘除行为会与预期完全相反——先看实例类型再排障（原文小技巧节已有此提醒）。
4. **客户端缓存导致感知延迟**：服务端摘了不等于调用方立刻知道。Eureka 是拉模式（30 秒拉一次 + 服务端只读缓存），Nacos 是推模式（订阅推送近实时，但客户端本地缓存兜底、注册中心全挂时仍用旧快照）。算感知延迟要把“服务端摘除”与“调用方拿到新表”分开算。
5. **默认值随版本变化**：Nacos 2.x 起客户端走 gRPC 长连接，连接活着即健康、断连即清临时实例，5 秒 HTTP 心跳是 1.x 模型；Eureka 在 Spring Cloud 新版本线里处于维护状态；Consul 的检查字段以所用版本官方文档为准。照抄旧博客的数字前先对版本。
6. **Consul 的注销不是默认行为**：`deregister-critical-service-after` 不配就永不自动注销，critical 服务会一直占着目录；Spring Cloud Consul 下对应的是 `health-check-critical-timeout`（文档示例 30m），两者别混着配。

### 来源（核验日期：2026-11-12）

- Eureka：Netflix/eureka 源码（master 分支，2026-11-12 核验）：`AbstractInstanceConfig.java`（`LEASE_RENEWAL_INTERVAL_SECONDS = 30`、`LEASE_EXPIRATION_DURATION_SECONDS = 90`）<https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/AbstractInstanceConfig.java>；`DefaultEurekaServerConfig.java`（`renewalPercentThreshold` 默认 0.85、`evictionIntervalTimerInMs` 默认 60000、`renewalThresholdUpdateIntervalMs` 默认 15×60×1000）<https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/DefaultEurekaServerConfig.java>；`DefaultEurekaClientConfig.java`（`getRegistryFetchIntervalSeconds` 默认 30）<https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/discovery/DefaultEurekaClientConfig.java>。自我保护叙述另见 DZone《The Mystery of Eureka Self-Preservation》<https://dzone.com/articles/the-mystery-of-eurekas-self-preservation>。
- Nacos：5/15/30 为客户端源码常量（Constants：`DEFAULT_HEART_BEAT_INTERVAL` 5s、`DEFAULT_HEART_BEAT_TIMEOUT` 15s、`DEFAULT_IP_DELETE_TIMEOUT` 30s），多篇源码分析一致，见 <https://programmer.group/microservice-analyze-the-source-code-the-health-check-of-nacos-is-so-simple.html>；Nacos 官网未在公开文档页逐项列出这三个默认值，此处标注为“已验证（源码常量）”而非“官方文档”。Nacos 官网 SDK 运行时指南（2.x gRPC 长连接、临时/持久实例语义）<https://www.nacos.io/en/docs/next/manual/user/sdk/runtime-guide/>；Nacos 官网概念页（不健康实例不返回给客户端、保护阈值）<https://nacos.io/en/docs/v2/concepts/>。
- Consul：HashiCorp 官方 Health check configuration reference（`interval` 必填无默认值、HTTP/TCP/gRPC 超时默认 10s、脚本 30s）<https://developer.hashicorp.com/consul/docs/reference/service/health-check>；Consul 旧版官方文档 Check Definition（`deregister_critical_service_after` 最小值 1 分钟、reaper 每 30 秒跑一次，镜像存档）<http://man.hubwiz.com/docset/Consul.docset/Contents/Resources/Documents/docs/agent/checks.html>，并经 v1.13.0 源码佐证（`agent/agent.go`：`CheckDeregisterIntervalMin` 下限钳制 + `reapServices` 按 `CheckReapInterval` 周期运行）<https://github.com/hashicorp/consul/blob/v1.13.0/agent/agent.go>；Spring Cloud Consul 官方附录（`health-check-critical-timeout` 示例 30m，2022.0.3 配置属性页实读核对）<https://docs.spring.io/spring-cloud-consul/docs/current/reference/html/appendix.html>。

事实分类：正文标注（已验证）的数字均来自上述来源；“最坏 180/210 秒”“Consul 示例约 100 秒”这类把多环节相加的上限为（推断），已在文中注明假设；“服务端只读缓存默认 30 秒”来自非官方源码分析、未在官方文档页核对到，标（待验证），不作为已验证事实使用。

## 小技巧与冷知识

【小技巧】Nacos 的服务变更是推过来的，Eureka 是拉过来的：Nacos 客户端订阅服务后，服务端有实例上下线会用 UDP 推送通知，感知几乎是实时的；Eureka 客户端默认每 30 秒才去注册中心拉一次注册表（`eureka.client.registry-fetch-interval-seconds` 默认 30）。排障时先分清你用的是哪种同步方式——Eureka 下“刚上线的实例半分钟内调不到”可能是正常的拉取间隔，不是注册失败。

【小技巧】Eureka 有自我保护模式，排障时别被它骗了：网络抖动导致心跳大量丢失时，Eureka 会进入自我保护，不再剔除长时间没心跳的实例，注册表里会留着一堆其实已经挂掉的实例。看到“实例明明下线了还在列表里”先看 Eureka 面板有没有自我保护的提示，别急着重启注册中心，网络恢复后心跳续上、保护模式自己会退出。

【小技巧】配置中心改了参数，Bean 不一定跟着变：Nacos Config 默认会把新配置推下来（`refresh-enabled` 默认 true），但只有加了 `@RefreshScope` 的 Bean 才会用新配置重建，没加的 Bean 读到的还是启动时的旧值。改完配置发现“配置中心显示新值、业务行为还是旧的”，先看目标 Bean 有没有 `@RefreshScope`，别先怀疑配置中心没下发。

【小技巧】Spring Cloud 2020.0 起 bootstrap 上下文默认关闭了：习惯写在 `bootstrap.yml` 里的 Nacos 配置（服务地址、namespace）会直接不生效，因为默认不再加载 bootstrap 上下文。想继续用 bootstrap.yml 要显式引入 `spring-cloud-starter-bootstrap` 依赖，否则把这些配置挪到 `application.yml` 里用 `spring.config.import` 引入；升级到 2021.0 线后“启动就连不上 Nacos”，先查这一条。

【小技巧】OpenFeign 默认不重试：`Retryer.NEVER_RETRY` 是 Feign 的默认行为，调用失败一次就直接抛异常，不会自动再试。想重试要显式配一个 `Retryer` Bean（或在配置里打开），而且重试只对幂等接口安全；原文第 6 节说“非幂等接口不要自动重试”，前提是你先确认当前到底有没有开重试——默认是没开的，别以为它在默默帮你重试。

【小技巧】Gateway 的路由是按顺序匹配、命中即停：路由表里先写具体的路径规则、后写通配兜底，顺序写反了兜底规则会把所有请求都吞掉，具体规则永远走不到。排查“某条路由配了却不生效”时，先看它在路由列表里的位置是不是被前面的宽泛规则截住了，路由匹配不是“最精确优先”，是“先命中优先”。

【小技巧】Nacos 的实例分临时与持久两种，选错了行为完全不同：临时实例（默认）靠心跳保活，掉线后会被注册中心摘掉；持久实例不靠心跳，断线后一直留在注册表里、靠健康检查探活。Dubbo 这类长连接服务常用持久实例，普通 HTTP 服务用临时实例；“服务下线了还在列表里”先看实例类型是不是持久的，别当成注册中心故障。

【冷知识】Nacos 的名字是三个词拼出来的：Na(ming) + Co(nfig) + S(ervice)，命名、配置、服务发现三件事正好对应它一件顶两件的定位——注册中心是 Naming，配置中心是 Config，再加上 Service 这层抽象。记住这个拆法，就不用再翻文档确认 Nacos 到底管哪几件事。

【冷知识】Spring Cloud 的版本命名在 2020.0 换过一次：之前用伦敦地铁站名做代号（Greenwich、Hoxton 这类），2020.0 起改成年份日历版本（2020.0、2021.0），代号从此退出。看老博客里的版本代号要先换算成年份版，再去对 Boot 的兼容表，别直接拿代号去搜最新文档。

【冷知识】Nacos 配置监听用的是长轮询，不是短轮询：客户端和服务端保持一条挂起的长连接，配置没变就一直等着、变了立刻返回，比每隔几秒轮询一次省连接也更快。这个机制也解释了为什么配置中心一挂客户端不会立刻报错——长连接超时后客户端会自己重连重试，配置读取在重试期间用的是本地缓存的旧值。

## 8. 面试与自测

- 注册中心挂了，存量服务还能互相调用吗？为什么？
- 客户端负载均衡与 Nginx 服务端均衡的区别在哪？
- 配置中心动态刷新是不是所有配置都适合？（想想数据源、连接池类配置）
- 单体拆微服务后，第一个爆出来的问题通常是什么？
- 什么信号说明一个系统「真的该拆了」？

- 注册中心里一个实例挂了，调用方最快/最慢多久不再调到它？Eureka、Nacos、Consul 有什么不同？

### 参考答案

1. 存量服务一般还能互相调用：调用方按服务名从注册中心查到的实例列表会在本地缓存，注册中心挂了，已缓存的存量实例地址仍可用。但新实例无法注册、上下线变化无法被发现，调用方只能用旧列表，扩缩容和故障摘除都会失效，所以只是「暂时能跑」，不是长期可依赖的状态。
2. 位置和决策方不同：客户端负载均衡（如 Spring Cloud LoadBalancer）由调用方自己从注册中心拿到的实例列表里按策略（轮询等）挑一台，直接调目标实例；Nginx 是服务端代理均衡，调用方只调 Nginx，由 Nginx 在后端实例间分发。简言之，前者是调用端选实例，后者是中间代理层选实例。
3. 不是所有配置都适合：只有能安全热替换的业务参数适合动态刷新。数据源、连接池这类配置涉及连接的创建与销毁，刷新往往不能直接生效，需要重建 Bean 或重启，否则可能出现连接泄漏、旧连接仍在用、新配置不生效等问题，改这类配置要按重启或重建流程走。
4. 通常是本地事务失效：一次业务操作跨进程后，原来的本地事务管不住了，要引入分布式事务的整套复杂度，或改造成最终一致性。同时排查也从一个日志文件变成跨服务链路追踪，超时、重试、幂等、限流熔断都从「不用考虑」变成「必须设计」，这些都是拆分后要立刻还的债。
5. 三类信号：某一模块（如影像处理、报表批处理）的资源消耗与主体差异巨大、需要独立扩缩容；多个团队改同一个工程、发版互相排队成为交付瓶颈；某部分有独立的合规或等保隔离要求、必须物理分开。没有这些信号时默认不拆，先在单体内划清模块边界、走模块化单体，真要拆也只拆一两个边界最清、交互最松的模块。

6. Eureka 心跳 30 秒、租约 90 秒，挂掉后服务端最坏 150 秒才摘除，调用方 30 秒拉一次表，最坏 180 秒还在调死实例；Nacos 心跳 5 秒、15 秒标不健康即不可见、30 秒摘除，感知半分钟量级；Consul 取决于检查间隔与 deregister。窗口越短，心跳与误判越多。

## 9. 关联知识

- [Spring-Boot](开发语言/Java/Spring/Spring-Boot.md)：单体基线；微服务组件都长在它上面。
- [分布式事务](数据库/分布式事务.md)：拆分后第一个要还的债。
- [限流熔断与降级](开发语言/Java/限流熔断与降级.md)：跨进程调用的自我保护。
