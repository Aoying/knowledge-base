---
知识点: OpenTelemetry
层级: 进阶
适用版本: OpenTelemetry Collector 0.165.x / Java instrumentation 2.32.x
更新日期: 2026-11-21
关联:
  - 监控告警
  - Prometheus与Grafana
  - Linux性能分析
  - JVM生产诊断工具链
标签:
  - 运维
  - 实践
  - 性能优化
---

# OpenTelemetry

一句话：OpenTelemetry（简称 OTel）是一套厂商中立的可观测性标准，规定了应用如何产生 Trace、Metric、Log 这三类遥测数据、以及用什么协议把它们送出去。

生活类比：它像快递行业的通用面单标准。以前每家快递公司用自己的面单，包裹一换公司就查不到；有了统一面单，不管包裹经过几家中转站，只凭一个单号就能全程追踪。OTel 就是给分布式系统定的那张“通用面单”。

## 基础知识

### 三大支柱与分工

系统出问题时，你会问三个不同的问题，三大支柱各答一个：

| 支柱 | 回答的问题 | 生活类比 | 典型数据 |
| --- | --- | --- | --- |
| Trace（链路追踪） | 这一次请求经过了哪些服务、每一步花了多久 | 快递单号：一个包裹经过哪些中转站、每站停留多久 | TraceID、Span、耗时 |
| Metric（指标） | 系统整体现在健康吗、趋势如何 | 汽车仪表盘：时速、油量、水温，一眼看整体状态 | QPS、延迟分位数、错误率 |
| Log（日志） | 当时到底发生了什么细节 | 行车记录仪：出事那一刻的画面和声音 | 时间戳、级别、消息、上下文 |

三者分工互补，不是互相替代：仪表盘告诉你车速异常（Metric），行车记录仪告诉你当时发生了什么（Log），快递单号告诉你包裹卡在哪一站（Trace）。只看一种，等于只用一只眼睛开车。

### Trace、Span、Resource 是什么

- **Trace**：一次完整请求的端到端轨迹，比如用户点一次“下单”，从网关到订单服务、库存服务、数据库的全部过程就是一条 Trace。
- **Span**：Trace 里的一段工作，比如“订单服务处理下单”是一个 Span，“查库存”又是另一个 Span。Span 之间是父子关系，拼起来就是一棵调用树。
- **Resource**：产生数据的资源身份，比如服务名 `order-service`、主机名、版本号。没有 Resource，出了问题你不知道是哪台机器、哪个版本在报错。
- **Attribute**：Span 或 Resource 上挂的键值对，比如 `http.method=POST`、`http.status_code=200`。相当于快递单上的收件人、重量、保价等字段。

一个 Span 长这样（简化后）：

```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "parentSpanId": "05e3ac9a4f6e8b8a",
  "name": "POST /orders",
  "startTime": "2026-11-16T10:00:00.123Z",
  "endTime": "2026-11-16T10:00:00.456Z",
  "attributes": { "http.status_code": 200 }
}
```

### 为什么需要统一标准

没有 OTel 之前，Java 用一套、Go 用一套、A 公司用 Jaeger、B 公司用 Zipkin，服务一换语言或一换后端，追踪就断了。OTel 把三件事统一了：**API**（各语言产生遥测数据的方式一致）、**SDK**（采样、批处理、导出等公共逻辑不用每个团队重写）、**OTLP 协议**（数据格式与端口全行业一致）。结果是：应用只管按标准产生数据，换后端（Jaeger、Tempo、厂商 SaaS）时改一行导出地址就行，代码不用动。

## 原理剖析

### TraceID 与 SpanID：快递单号是怎么编的

- **TraceID**：128 位，写成 32 个十六进制字符。同一条 Trace 里所有 Span 共用一个 TraceID，就像同一个包裹在所有中转站共用一个单号。
- **SpanID**：64 位，写成 16 个十六进制字符。每个 Span 各有一个唯一的 SpanID，相当于包裹在每个中转站的“本站操作编号”。
- **ParentSpanID**：子 Span 记录父 Span 的 SpanID，父子关系就是这样串起来的。

TraceID 是随机生成的，128 位意味着碰撞概率低到可以忽略：哪怕每秒产生 10 万条 Trace，连续跑 100 年，撞号的概率仍然远低于中一次彩票头奖，实际工程里直接当成全局唯一用。

### 上下文传播：单号怎么跟着包裹走

服务 A 调服务 B 时，要把 TraceID 传过去，否则 B 产生的 Span 就成了一条孤儿链路。OTel 用 W3C 标准 `traceparent` 请求头传递，格式分四段：

```text
00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
│  │                                │                │
│  │                                │                └─ flags：01 表示已采样，00 表示未采样
│  │                                └─ parent-id：父 Span 的 SpanID（16 个十六进制字符）
│  └─ trace-id：整条链路的 TraceID（32 个十六进制字符）
└─ version：版本号，目前是 00
```

逐步拆解上面这个例子：

1. `00`：版本号，告诉接收方按第 0 版规则解析。
2. `4bf92f3577b34da6a3ce929d0e0e4736`：TraceID，32 个字符，整条链路共用。
3. `00f067aa0ba902b7`：父 SpanID，16 个字符，说明调用方是哪个 Span。
4. `01`：flags，`01` 表示这条 Trace 已被采样、要保留；`00` 表示不保留。

传播过程像接力赛交接棒：网关收到请求时生成 TraceID，调订单服务时把 `traceparent` 放进 HTTP 请求头；订单服务解析出 TraceID，新建一个子 Span，再调库存服务时把更新后的 `traceparent`（parent-id 换成自己的 SpanID）继续往下传。每个服务只做“读出旧棒、写上自己名字、传给下一棒”这三步。

下面这张时序图展示一次下单请求的完整调用链：

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant G as API 网关
    participant O as 订单服务
    participant S as 库存服务
    participant D as 数据库
    U->>G: 下单请求，生成 TraceID
    G->>O: HTTP 调用，带 traceparent 请求头
    O->>S: 查库存，parent-id 换成订单 Span
    S->>D: SQL 查询库存，继续透传 TraceID
    D-->>S: 返回库存余量
    S-->>O: 返回扣减结果
    O-->>G: 返回订单号
    G-->>U: 返回下单成功
```

图中每一步箭头都带着同一个 TraceID，只是 parent-id 逐级更换——这就是“上下文传播”的全部含义。

### Collector 架构：Receiver / Processor / Exporter 三段式

Collector 是 OTel 的数据中转站，应用不直连后端，先把数据发给 Collector，再由它统一处理、转发。它的内部是三段式流水线：

- **Receiver（收）**：负责接数据。OTLP Receiver 默认监听 4317（gRPC）和 4318（HTTP）两个端口；还能接 Prometheus 拉取、文件日志、主机指标等多种来源。像快递分拣中心的卸货口，不同车队从不同口进。
- **Processor（处理）**：负责加工数据。`batch` 处理器把零散的 Span 攒成一批再发，减少网络请求次数；`attributes` 处理器给数据打标签或脱敏，比如删掉手机号字段；`memory_limiter` 处理器在内存快满时限流，防止 Collector 自己被撑爆。
- **Exporter（送）**：负责发数据。OTLP Exporter 发给 Jaeger、Tempo 等后端；Prometheus Exporter 把指标暴露给 Prometheus 拉取；Logging Exporter 打印到控制台，一般只用来调试。

数据流向是单向的：Receiver → Processor → Exporter，三段之间用管道（pipeline）连接，一类信号（traces、metrics、logs）配一条管道。

下面这张架构图展示数据从应用流进 Collector、再流向后端的完整路径：

```mermaid
flowchart LR
    subgraph 应用侧
        A[Java 应用 OTel Agent]
        B[Go 应用 OTel SDK]
    end
    subgraph Collector
        R1[OTLP Receiver 4317 gRPC]
        R2[OTLP Receiver 4318 HTTP]
        P1[batch 处理器 攒批发送]
        P2[attributes 处理器 打标签脱敏]
        E1[OTLP Exporter 发往后端]
        E2[Prometheus Exporter 供拉取]
    end
    subgraph 后端
        J[Jaeger 或 Tempo 存 Trace]
        P[Prometheus 存 Metric]
    end
    A --> R1; B --> R2
    R1 --> P1; R2 --> P1
    P1 --> P2
    P2 --> E1; P2 --> E2
    E1 --> J; E2 --> P
```

一个最简可用的 Collector 配置只要三段各配一个组件：

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch: {}
exporters:
  otlp:
    endpoint: jaeger:4317
    tls:
      insecure: true
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

读这份配置只要记住一句话：`service.pipelines.traces` 这一节就是把上面三段用名字串起来，告诉 Collector“Trace 从 otlp 收进来、过 batch 处理、从 otlp 发出去”。

### 部署形态：Agent 还是 Gateway

- **Agent 模式**：每台机器或每个 Pod 旁跑一个 Collector，只收本机数据再转发给中心。优点是本机缓存、断网续传；缺点是数量多、难统一升级。
- **Gateway 模式**：中心部署少数几个 Collector，所有 Agent 先发给它，再由它统一做采样、脱敏、路由后发往后端。优点是集中处理；缺点是多了一跳、要做多副本防单点。常见做法是两者结合：Agent 做轻量接收和重试，Gateway 做尾部采样和统一出口；小型系统可以直接只用 Gateway，省掉一层运维成本。

## 进阶知识

### 采样策略与成本：一笔算清楚的账

Trace 数据量极大，不采样根本存不起。假设一个中等规模系统每秒产生 1 万条 Span，全量保留会发生什么？下面用 Python 复算过的一组数字逐步算给你看。

**第一步：一天产生多少条？** 每秒 10,000 条 × 60 秒 = 每分钟 600,000 条；每分钟 600,000 条 × 60 分钟 = 每小时 36,000,000 条；每小时 36,000,000 条 × 24 小时 = 每天 864,000,000 条。一天 8.64 亿条 Span，全量保留一个月就是约 259 亿条。

**第二步：按 1% 采样后剩多少？** 每秒保留 10,000 × 1% = 100 条；每天保留 100 × 86,400 秒 = 8,640,000 条；每天丢弃 864,000,000 - 8,640,000 = 855,360,000 条。采样后每天只剩 864 万条，是全量的百分之一。

**第三步：换算成存储。按每条 Span 平均 1 KiB（1024 字节）估算：** 全量每天 864,000,000 × 1 KiB ≈ 824 GiB；采样后每天 8,640,000 × 1 KiB ≈ 8.24 GiB；每天省下 824 - 8.24 ≈ 816 GiB；每月全量 824 × 30 ≈ 24,719 GiB，约 24.1 TiB；每月采样后 8.24 × 30 ≈ 247 GiB。

**第四步：换算成钱。按对象存储 0.023 美元/GiB/月估算（只算当月增量）：** 全量月费用 24,719 × 0.023 ≈ 568.54 美元；采样后月费用 247 × 0.023 ≈ 5.69 美元；每月省下约 562.85 美元。

**第五步：换个口径再验证。保留 7 天滚动窗口时：** 全量 7 天窗口 824 × 7 ≈ 5,768 GiB，约 5.63 TiB；采样后 7 天窗口 8.24 × 7 ≈ 57.68 GiB。

结论：1% 采样把每天 824 GiB 的写入压到 8.24 GiB，7 天窗口从 5.63 TiB 压到 57.68 GiB。采样率每翻一档，成本跟着翻一档：10% 采样每天是 86,400,000 条、约 82.4 GiB；5% 采样每天是 43,200,000 条、约 41.2 GiB。

**换算成 Trace 更直观：** 如果平均每条 Trace 由 20 个 Span 组成，每秒 1 万条 Span 就是每秒 500 条 Trace，全量每天 43,200,000 条 Trace，1% 采样后每天剩 432,000 条——排查问题完全够用。

### 头部采样与尾部采样

- **头部采样（Head Sampling）**：请求刚进来就按比例决定留不留，比如 1% 的请求全程记录。优点是简单、开销小；缺点是决定时还不知道这次请求会不会出错，报错的请求可能正好没被采上。
- **尾部采样（Tail Sampling）**：等整条 Trace 跑完再决定留不留，只留出错的、特别慢的、或带特定标签的。优点是“好钢用在刀刃上”，出错的请求一条不落；缺点是 Collector 要先把全量 Trace 攒在内存里再筛选，内存和复杂度都更高。
- **父子一致采样**：一旦父 Span 决定采样，整条链路所有子 Span 都采样；反之全都不采。靠 `traceparent` 里的 flags 位（`01` 或 `00`）实现。这是保证 Trace 完整不断链的关键，缺了它，链路会像被撕碎的快递单一样拼不起来。

生产建议：入口处用 1%～5% 的头部采样兜底，再在 Gateway 层加尾部采样规则——错误 Trace 100% 保留、慢于 1 秒的 Trace 100% 保留。这样既省了 99% 的存储，又不丢任何一次故障现场。

### Metric 的三种类型

- **Counter（计数器）**：只增不减的累计值。超市收银台的累计收款额就是 Counter，今天收了 3 万，明天只会更多，不会倒退。适合统计请求总数、错误总数。
- **Gauge（仪表值）**：可增可减的瞬时值。停车场剩余车位数就是 Gauge，有车进就减、有车出就加。适合统计当前连接数、内存占用、队列深度。
- **Histogram（直方图）**：把数值按区间分桶统计。超市统计“顾客排队时长”时分桶：0～1 分钟多少人、1～3 分钟多少人、3 分钟以上多少人。适合统计请求延迟分布，算 P50、P95、P99 就靠它。

一条铁律：**不要把 userId、订单号这种高基数（取值成千上万）的字段放进 Metric 标签**。每个标签值组合都会生成一条独立的时间序列，10 万个用户就是 10 万条序列，Prometheus 会被撑爆。高基数信息放 Trace 的 Attribute 里查，不要放 Metric 标签里。

### 与 Prometheus 的互补关系

OTel 和 Prometheus 不是二选一，是分工合作：

- **指标归 Prometheus**：QPS、延迟分位数、错误率、资源用量，这些“看趋势、做告警”的数据由 Prometheus 存、由 PromQL 查、由 Grafana 画图。Prometheus 的拉取模型和告警生态在这里是主场。
- **链路归 OTel**：单次请求经过了哪些服务、哪一步慢、哪一步报错，这些“看个案、做定位”的数据由 OTel 的 Trace 负责，存在 Jaeger 或 Tempo 里。
- **两者衔接**：OTel Collector 可以把 Metric 通过 Prometheus Exporter 或 Remote Write 送进 Prometheus；反过来，Grafana 里可以从一张 Prometheus 告警图一键跳到同一时间段的 Trace 做下钻。正确姿势是“Prometheus 看面、OTel 看点”，面有点结合才定位得快。

## 采样与成本定量

「进阶知识」一节已经算过一笔账：10,000 spans/s、每条 Span 1 KiB、1% 采样，每天从约 824 GiB 压到约 8.24 GiB。本节把那笔账往深里算三步：先认清 SDK 自带的最常用的五种采样器该怎么选，再算单条 Span 的体积到底由什么决定，最后算 Collector 里攒批与尾采缓冲的内存代价。采样率本身不是目的，目的是用最少的钱留住最值得查的 Trace。

### 原理拆解

#### 最常用的五种内置采样器：各管什么

OpenTelemetry 规范定义了多组内置采样器，最常用的是下面五种（新版规范另有 `ProbabilitySampler`——按 W3C Trace Context Level 2 的 56 位随机性做一致性概率采样，是 `TraceIdRatioBased` 的推荐替代品——以及 `AlwaysRecord`、`CompositeSampler`，本节不展开）：

| 采样器 | 决定逻辑 | 环境变量取值 | 适用场景 |
| --- | --- | --- | --- |
| `AlwaysOn` | 全部保留 | `always_on` | 开发联调、故障复现，需要完整链路时 |
| `AlwaysOff` | 全部丢弃 | `always_off` | 只想保留埋点、不想出任何 Trace 时 |
| `TraceIdRatioBased` | 对 TraceID 做确定性哈希，按比例保留 | `traceidratio` | 生产环境按固定比例采样 |
| `ParentBased` | 有父 Span 跟随父级决定，无父 Span 用根采样器决定 | `parentbased_always_on`、`parentbased_always_off`、`parentbased_traceidratio` | 分布式系统，保证整条链路要么全留、要么全丢 |
| `JaegerRemote` | 从远端（Jaeger）拉取采样策略，定时轮询更新 | `jaeger_remote`、`parentbased_jaeger_remote` | 需要在不重启服务的情况下集中调采样率时 |

默认行为要记牢：**规范默认采样器是 `ParentBased(root=AlwaysOn)`，即不配置时 100% 采样**。原文实战案例里用环境变量配成 `parentbased_traceidratio` 加 `0.01`，就是把这个默认的 100% 改成 1%。

`TraceIdRatioBased` 有两个关键性质。第一，决定只依赖 TraceID：同一条 Trace 在各个服务上独立算出相同结论，不用服务之间互相商量，这是链路不断链的数学基础。第二，它的哈希算法在规范里尚未最终定死，不同语言 SDK、甚至同一 SDK 的不同版本，对同一条 Trace 可能算出不一致的结果；所以规范建议只把它当作 `ParentBased` 的根采样器用，不要裸用。

`ParentBased` 区分五种父级状态：无父级（根 Span）、远端父级已采样、远端父级未采样、本地父级已采样、本地父级未采样。跟随父级决定的部分靠 `traceparent` 请求头里的 flags 位（`01` 表示已采样、`00` 表示未采样）传递，「原理剖析」一节已讲过这个头的格式。

#### 头采样与尾采样：取舍不在“哪个好”，在“什么时候决定”

「进阶知识」一节只讲了两者定义，这里补一张对照表，重点看各自省了什么、付了什么：

| 维度 | 头采样 | 尾采样 |
| --- | --- | --- |
| 决定时点 | 根 Span 启动时，请求还没跑 | 整条 Trace 收齐后，在 Collector 里决定 |
| 决定时能看到什么 | 只有 TraceID 和根 Span 属性 | 全部 Span、状态码、端到端耗时、全部属性 |
| 状态 | 无状态，每个服务独立决定 | 有状态，Collector 要把在途 Trace 全攒在内存里 |
| 省掉的成本 | 应用侧出口成本也省：没采样的 Span 根本不产生、不序列化、不发送 | 只省后端存储成本；应用侧全量产生、发送的成本一分不少 |
| 盲区 | 看不到结果：报错和慢请求与健康请求被同等概率丢掉 | 看得到结果：错误、慢请求可以 100% 保留 |
| 典型故障 | 稀有报错正好落在没采样的 99% 里，事后查不到 | 内存撑爆、迟到的 Span 赶不上决定、同一条 Trace 的 Span 落到不同 Collector 实例上 |

生产环境的通行做法是两者叠加：入口先用头采样把明显过量的流量削掉，再在 Gateway 层用尾采样规则兜底——错误 Trace 100% 保留、慢 Trace 100% 保留、其余按低比例保留基线。原文「进阶知识」一节末尾的生产建议就是这个思路。

尾采样在 Collector 里由 `tail_sampling` 处理器实现，关键参数都有默认值（以 contrib v0.148.0 的 README 为准）：`decision_wait` 默认 30 秒，指从收到一条 Trace 的第一个 Span 起、等多久就强制做决定；`num_traces` 默认 50000，指内存里最多同时攒多少条 Trace；`expected_new_traces_per_sec` 默认 0，用于预分配内部结构；`policies` 没有默认值，必须配。策略之间是“或”关系：任意一条策略说保留，这条 Trace 就保留。常见写法是三条策略叠加：状态码为 `ERROR` 的保留、耗时超过阈值（例如 2000 毫秒）的保留、其余按 5% 或 10% 的概率保留基线。

#### 单条 Span 的体积量级：500 B 到 2 KiB 是常态

一条 Span 在 OTLP 协议里的体积通常在 500 B 到 2 KiB 之间，具体取决于挂了多少属性；一条 50 个 Span 的 Trace 就是 25 KB 到 100 KB。落盘后因为压缩，体积还会再缩：压缩后每条 Span 通常在 300 B 到 600 B 之间，一条 1 KiB 的 Span 压缩后约 250 B，OTLP 的压缩比约 4 比 1。也就是说，体积由两样东西决定：Span 数量（调用链有多深）和每条 Span 的属性多少。「疑难杂症」一节的「属性爆炸」讲过，往 Span 上挂请求体全文、完整 SQL 文本这类大字段，会让单条 Span 的体积直接翻几倍；Agent 场景更极端，带提示词与模型输出的 Span 常见 20 KB 到 30 KB 一条。

### 手算示例

下面三组手算用同一个新场景：**每秒 5,000 条 Span，平均每条 Trace 由 20 个 Span 组成**（即每秒 250 条 Trace）。场景与原文「进阶知识」一节的 10,000 spans/s 不同，避免重复。

#### 手算 1：日均 Span 量 × 单 Span 体积 → 日增量

- 日 Span 量：5,000 × 86,400 秒 = 432,000,000 条。
- 按 OTLP 线上体积 500 B/条：432,000,000 × 500 B = 216 GB，约 201 GiB。
- 按属性多的 2 KiB/条：432,000,000 × 2,048 B = 884.7 GB，约 824 GiB。
- 按压缩后落盘 420 B/条（取 300–600 B 区间中值）：432,000,000 × 420 B = 181.4 GB，约 169 GiB。

结论：单条 Span 体积差 4 倍，日增量就差 4 倍。**控体积与降采样率是同一笔账的两边**：把 SQL 全文换成模板、删掉大字段，效果相当于把采样率直接砍到四分之一。

#### 手算 2：采样率从 100% 降到 1%，30 天保留的成本变化

沿用原文的单价 0.023 美元/GiB/月，但改按 30 天稳态保留口径算（原文按当月增量口径，两个口径结论方向一致）：

- 100% 采样：每天落盘约 169 GiB，30 天稳态保留 169 × 30 = 5,069 GiB，约 4.95 TiB；月费 5,069 × 0.023 ≈ 116.60 美元。
- 1% 采样：每天落盘约 1.69 GiB，30 天稳态保留约 50.7 GiB；月费 50.7 × 0.023 ≈ 1.17 美元。
- 省下的钱：116.60 - 1.17 ≈ 115.43 美元/月，降幅 99%。

如果改用混合尾采（错误 100% 保留、慢请求 100% 保留、其余 1% 基线），先算有效保留率。假设错误率 0.5%、慢请求占 1%、两者不重叠：

- 有效保留率 = 0.5% + 1% + 1% × (1 - 0.5% - 1%) = 2.485%。
- 每天保留 432,000,000 × 2.485% = 10,735,200 条 Span，每天落盘约 4.2 GiB，30 天稳态保留约 126 GiB，月费约 2.90 美元。
- 只比纯 1% 头采样多花约 1.73 美元/月，换来的却是错误与慢请求一条不落。**这就是尾采的性价比：多花 2.5 倍的存储，买断了“查不到报错 Trace”的风险。**

#### 手算 3：Collector 攒批与尾采缓冲的内存/延迟权衡

先算 `batch` 处理器。默认值（以 Collector v0.120.0 的 README 为准）：`send_batch_size` 默认 8192，`timeout` 默认 200 毫秒，`send_batch_max_size` 默认 0（即不设上限）。注意 `send_batch_size` 是触发发送的条数，不是每批的上限；要硬上限需另配 `send_batch_max_size`。按每秒 5,000 条 Span、每条 500 B 算：

- 默认配置：200 毫秒超时先触发，每批约 5,000 × 0.2 = 1,000 条 Span，约 0.5 MB（仅 Span 原始体积口径，不含 Go 对象开销），每秒发约 5 次，导出延迟最高 200 毫秒。
- 把 `send_batch_size` 保持 8192、`timeout` 放到 5 秒：8192 条约 1.64 秒攒满，每批约 4.1 MB，每秒约发 0.61 次，请求数降到约八分之一，但导出延迟最高 1.64 秒，内存占用涨到约 8 倍。
- 把 `send_batch_size` 改成 1024（「疑难杂症」一节建议的起点）：约 0.205 秒攒满，每批约 0.51 MB，每秒约发 5 次，与默认配置接近。

结论：批越大，请求越少、压缩越好，但内存占用与导出延迟跟着涨；批越小，延迟越低，但请求越碎。低流量服务上超时先触发、高流量服务上条数先触发，调参之前先看清是哪一个在起作用。

再算尾采缓冲。估算公式是 `num_traces` 约等于每秒新增 Trace 数 × `decision_wait` × 1.2，内存约等于平均每条 Trace 的体积 × `num_traces`：

- 本节场景：每秒 250 条 Trace，`decision_wait` 设 10 秒，平均每条 Trace 20 个 Span × 500 B = 10 KB。缓冲 Trace 数约 250 × 10 × 1.2 = 3,000 条，内存约 3,000 × 10 KB ≈ 30 MB。
- 对照一个已公开的估算例：每秒 10,000 条 Trace、等 15 秒、每条 Trace 10 KB，缓冲约 180,000 条 Trace、内存约 1.8 GB。

结论：尾采的内存账是线性的，流量翻 40 倍、等待时间翻 1.5 倍，内存就从 30 MB 涨到 1.8 GB。`num_traces` 与 `decision_wait` 不是随便填的两个数，填之前先用上面这个公式算一遍。

### 边界条件

- `decision_wait` 必须长于最慢的 Trace。单体应用 5 到 10 秒够用，微服务 15 到 20 秒起，跨地域调用 30 秒以上；等的时间一到，哪怕尾部 Span 还没到，也按已收到的 Span 做决定，迟到的 Span 会被丢弃。
- 同一条 Trace 的全部 Span 必须落到同一个 Collector 实例上，否则尾采处理器看不到完整 Trace。Gateway 层多副本时，要在前一层用 `loadbalancing` 导出器按 TraceID 路由，或用无头服务（headless service）让各副本直接收到按 TraceID 分流的流量。
- 处理器顺序有讲究：`batch` 应放在 `memory_limiter` 与采样处理器之后，先限流、先采样丢弃，再攒批发送；反过来会把本该丢掉的数据也攒进批里。
- 版本边界：本文适用的 Collector 是 0.165.x，`batch` 处理器在该版本仍是标准做法。但官方已宣布弃用 `batch` 处理器，改为导出器级批量（exporter-level batching，计划在 v0.171.0 起成为多数导出器的默认行为，约 2027 年 2 月），`batch` 处理器本身计划在 v0.177.0（约 2027 年 5 月）移除，替代品是新的 `queue_batch` 处理器。升级到 0.171.x 之后应按官方迁移指南去掉管道里的 `batch` 处理器。
- 采样率可以反推，不用拍脑袋：月存储预算 ÷（每条 Span 落盘字节数 × 保留天数 × 86,400）= 能承受的每秒 Span 数上限，再用上限 ÷ 峰值每秒 Span 数得到采样率。量纲上注意用压缩后落盘字节数，不用线上体积，否则会把需求高估约 4 倍。

### 反例与易错点

- **裸用 `traceidratio`，不用 `ParentBased` 包一层**：每个服务各自按自己的决定采样，同一条 Trace 在这个服务被保留、在那个服务被丢弃，链路出现断洞。正确做法永远是 `parentbased_traceidratio`，让下游跟随上游的决定。
- **用“把采样率调到 100%”解决查不到报错 Trace**：成本账在手算 2 里，100% 采样每月约 116.60 美元，1% 只要约 1.17 美元。正确做法是上尾采，错误 Trace 100% 保留，基线保持 1%。
- **以为尾采能省应用侧成本**：尾采只省后端存储，应用侧仍要全量产生、序列化、发送 Span，出口流量与 SDK 开销一分不少。想连应用侧一起省，只能靠头采样。
- **把 `send_batch_size` 当成每批上限**：它是触发发送的条数，攒够就发，但不限制一批最多多大；后端要求单次请求有硬上限时，必须另配 `send_batch_max_size`。
- **只算采样率、不算 Span 体积**：采样率从 100% 降到 1% 省了 99%，但往 Span 上多挂一个 2 KB 的 SQL 全文，体积直接翻 4 倍。手算 1 的两档体积就是证明：先瘦身，再谈采样率。
- **尾采缓冲按默认值硬上生产**：`num_traces` 默认 50000、`decision_wait` 默认 30 秒，在高流量下内存可能直接翻到 GB 级。先用手算 3 的公式按自己的流量算，再填数；同时配好 `memory_limiter`，宁可限流丢数据，也别让 Collector OOM。
- **混用 `AlwaysOn` 的服务**：上游用 `parentbased_traceidratio` 决定丢弃，下游某个服务配了 `AlwaysOn`，会无视父级决定全量保留，链路两端采样不一致，排查时看到一半 Trace、另一半凭空消失。

### 来源

核验日期 2026-11-21，全部链接为核验当日可访问的原文链接：

- 采样器定义与默认采样器 `ParentBased(root=AlwaysOn)`、内置采样器（最常用的五种）：OpenTelemetry 规范 Tracing SDK（Sampling 一节），`https://raw.githubusercontent.com/open-telemetry/opentelemetry-specification/main/specification/trace/sdk.md`
- 采样器环境变量取值与默认行为（默认 100% 采样）：OpenTelemetry 官方文档 Sampling（.NET / Erlang），`https://opentelemetry.io/ko/docs/languages/dotnet/sampling/`、`https://opentelemetry.io/fr/docs/languages/erlang/sampling/`
- 头采样与尾采样的取舍对照：`https://blog.oneenv.io/articles/head-vs-tail-sampling-designing-a-hybrid-trace-sampling-strategy`、`https://www.distributed-tracing.com/distributed-tracing-fundamentals-architecture/choosing-between-head-based-and-tail-based-sampling/`、`https://dev.to/uptrace/opentelemetry-sampling-head-based-and-tail-based-explained-4pm7`
- `tail_sampling` 处理器参数默认值（`decision_wait` 30 秒、`num_traces` 50000、策略为“或”关系）与内存估算例：`https://raw.githubusercontent.com/open-telemetry/opentelemetry-collector-contrib/v0.148.0/processor/tailsamplingprocessor/README.md`、`https://dzone.com/articles/tail-based-opentelemetry-collector-sampling`、`https://www.datadoghq.com/blog/control-trace-volume-with-opentelemetry-tail-based-sampling/`
- `batch` 处理器默认值（`send_batch_size` 8192、`timeout` 200 毫秒、`send_batch_max_size` 0）与弃用迁移时间表：`https://raw.githubusercontent.com/open-telemetry/opentelemetry-collector/v0.120.0/processor/batchprocessor/README.md`、`https://opentelemetry.io/blog/2026/collector-batching-migration/`
- 单条 Span 体积量级（OTLP 线上 500 B–2 KiB、压缩后 300–600 B/条、压缩比约 4 比 1）与采样率反推公式：`https://www.kunwar.page/chapter/095-distributed-tracing-opentelemetry-jaeger-tempo`、`https://www.distributed-tracing.com/distributed-tracing-fundamentals-architecture/choosing-between-head-based-and-tail-based-sampling/calculating-sampling-rates-for-a-trace-budget/`

## 疑难杂症

### Trace 断链：下游的 Span 拼不上来

- **现象与原因**：Jaeger 里一条 Trace 只有网关和订单服务，库存服务的 Span 凭空消失，或者出现两条各只有一半的 Trace。最常见的是中间经过了不透传请求头的环节，比如用了自定义的 HTTP 客户端没注入 `traceparent`，或者走了消息队列、定时任务，上下文没跟着消息走。
- **解法**：先查调用链上每个环节是否透传了 `traceparent` 请求头；消息队列场景用 OTel 的消息生产/消费埋点，把上下文写进消息属性里再透传；定时任务入口手动创建一个根 Span。

### 查不到某次报错的 Trace：采样把它丢了

- **现象与原因**：用户反馈某笔订单失败，拿着订单号去 Jaeger 里搜，什么都没有。头部采样按概率丢弃，报错的那次请求正好落在没被采样的 99% 里。
- **解法**：上尾部采样，错误 Trace 100% 保留；或者在业务代码里对关键操作手动标记——出错时把采样标志改成保留。不要靠“把采样率调到 100%”解决，成本账前面已经算过，撑不住。

### Collector 背压与丢数据：高峰期数据进不来

- **现象与原因**：大促高峰期，Trace 数量不升反降，后端查到的量比应用发出的量少。后端写入变慢，Exporter 发不出去，数据在 Collector 内部队列里越积越多，内存满了就开始丢；或者 `batch` 处理器攒批太大，单次发送超时。
- **解法**：给 Collector 配 `memory_limiter` 处理器，内存到阈值先限流而不是直接 OOM；Exporter 开重试和发送队列；`batch` 的 `send_batch_size` 别设太大，先设 1024 再按需调；Gateway 层做多副本分摊压力。

### 时钟偏差：Span 时间对不上

- **现象与原因**：一条 Trace 里子 Span 的开始时间比父 Span 还早，调用关系图画出来是乱的。各台机器的系统时钟没对齐，差了几百毫秒甚至几秒。
- **解法**：所有机器开 NTP 对时；跨机房部署时尤其要检查，时钟偏差超过 100 毫秒就会让调用时序失真。Span 之间的父子关系靠 SpanID，不靠时间戳，关系不会断，但时序图会难看。

### 属性爆炸：标签打得太多太碎

- **现象与原因**：Collector 内存一路上涨，后端存储增长远超预期。Span 上挂了请求体全文、SQL 完整文本、用户身份证号这类大字段或高基数字段。
- **解法**：Attribute 只保留定位问题需要的字段；敏感字段用 `attributes` 处理器脱敏或删除；SQL 文本只保留模板（把参数替换成问号），不要存完整 SQL。

## 实战案例

### Java 应用接入：两种方式

**方式一：Java Agent 零代码接入。** 不改一行业务代码，启动命令上加一个 `-javaagent` 参数，Agent 会在类加载时自动给 Spring MVC、JDBC、HTTP 客户端等常见框架织入埋点。

```bash
java -javaagent:/opt/otel/opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
  -jar order-service.jar
```

也可以用环境变量代替 `-D` 参数，容器部署时更方便：

```bash
export OTEL_SERVICE_NAME=order-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.01
java -javaagent:/opt/otel/opentelemetry-javaagent.jar -jar order-service.jar
```

上面两段命令里的 `OTEL_TRACES_SAMPLER_ARG=0.01` 就是前面算账用的 1% 采样率。`parentbased_traceidratio` 的意思是：有父 Span 就跟随父级的采样决定，没有父级就按 1% 概率采样，保证整条链路要么全留、要么全丢。

> 说明：本机没有安装 otelcol 和 Java Agent 环境，上述命令按 OpenTelemetry 官方文档写法给出，未在本地实测；输出示例为按官方文档给出的典型示例。接入后可在 Collector 日志或 Jaeger 界面确认数据是否到达。

**方式二：手动埋点。** Agent 覆盖不到的业务逻辑（比如一段核心计价算法）需要手动创建 Span：

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

public class PricingService {
    private static final Tracer tracer = GlobalOpenTelemetry.getTracer("order-service", "1.0.0");
    public long calculatePrice(Order order) {
        Span span = tracer.spanBuilder("calculatePrice")
            .setAttribute("order.id", order.getId())
            .setAttribute("order.itemCount", order.getItems().size())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            long price = doCalculate(order);
            span.setAttribute("order.totalPrice", price);
            return price;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(io.opentelemetry.api.trace.StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

手动埋点记住三件事：`try (Scope scope = span.makeCurrent())` 让子 Span 能找到父 Span；出错时 `recordException` 加 `setStatus(ERROR)`，否则 Jaeger 里看不出这段报错了；`span.end()` 必须在 `finally` 里调，漏调等于这段 Span 永远发不出去。

### 一次慢请求定位：从 Trace 找到慢 SQL

某天订单接口 P99 从 200 毫秒涨到 2 秒，排查过程只用了三步：

1. **Metric 发现异常**：Prometheus 图上 `http.server.request.duration` 的 P99 曲线在 10:00 开始抬头，确认是真实劣化，不是单次抖动。
2. **Trace 定位环节**：在 Jaeger 里按时间段找一条慢 Trace，调用树显示网关 20 毫秒、订单服务 30 毫秒、库存服务 40 毫秒，剩下的 1.9 秒全耗在“查订单明细”这一步的数据库 Span 上。
3. **Log 确认细节**：用这条 Trace 的 TraceID 去日志系统搜同一时间段的日志，看到数据库连接池等待日志刷屏，最终确认是连接池打满导致 SQL 排队。

三步分工正好对应三大支柱：Metric 发现面上的异常，Trace 定位到具体的慢环节，Log 给出当时的细节证据。任何一步缺了，排查都要多绕几个弯。

## 面试与自测

1. Trace、Metric、Log 三大支柱分别回答什么问题？为什么说三者是互补而不是互相替代？
2. TraceID、SpanID、ParentSpanID 各是什么？服务 A 调用服务 B 时，TraceID 是怎么传过去的？
3. 每秒产生 1 万条 Span、按 1% 采样，一天后保留多少条？如果每条 Span 平均 1 KiB，全量和采样后每天各需要多少存储？
4. 一个系统已经接了 Prometheus，为什么还需要接 OpenTelemetry？两者应该怎么分工？

5. 生产环境把 SDK 采样器从默认改成 `parentbased_traceidratio`（0.01）后，某条报错的 Trace 还是查不到，为什么？应该如何配置才能保证错误 Trace 100% 保留？已知每秒新增 250 条 Trace、`decision_wait` 设 10 秒、平均每条 Trace 10 KB，估算尾采缓冲需要多少内存？

### 参考答案

1. Trace 回答“这一次请求经过了哪些服务、每一步花了多久”，Metric 回答“系统整体现在健康吗、趋势如何”，Log 回答“当时到底发生了什么细节”。三者互补：Metric 发现面上的异常、Trace 定位到具体的慢环节或错环节、Log 给出当时的细节证据；只用一种等于只用一只眼睛开车，排查要多绕弯。
2. TraceID 是 128 位（32 个十六进制字符），整条 Trace 共用；SpanID 是 64 位（16 个十六进制字符），每个 Span 唯一；ParentSpanID 记录父 Span 的 SpanID，用来串起父子关系。跨服务传递靠 W3C `traceparent` 请求头，格式为 `版本-TraceID-父SpanID-flags`，调用方把更新后的请求头放进 HTTP 请求，接收方解析后新建子 Span 并继续往下传。
3. 每秒保留 10,000 × 1% = 100 条，一天保留 100 × 86,400 = 8,640,000 条，丢弃 855,360,000 条。按每条 1 KiB 算，全量每天 864,000,000 × 1 KiB ≈ 824 GiB，采样后每天约 8.24 GiB，每天省约 816 GiB；每月全量约 24.1 TiB、费用约 568.54 美元（按 0.023 美元/GiB/月），采样后每月约 247 GiB、费用约 5.69 美元。
4. Prometheus 擅长指标：QPS、延迟分位数、错误率、资源用量，适合看趋势和做告警；OpenTelemetry 擅长链路：单次请求经过了哪些服务、哪一步慢、哪一步报错，适合做个案定位。正确分工是“指标归 Prometheus、链路归 OTel”：Collector 把 Metric 送进 Prometheus，把 Trace 送进 Jaeger 或 Tempo，Grafana 里从告警图一键下钻到 Trace。

5. 头采样在根 Span 启动时就决定留不留，那时请求还没跑、不知道会不会报错，所以报错请求与健康请求被同等概率丢掉：1% 采样下 99% 的报错 Trace 根本不会产生。正确做法是在 Gateway 层加 `tail_sampling` 处理器，配三条策略（策略之间是“或”关系）：状态码为 `ERROR` 的 100% 保留、耗时超过阈值（例如 2000 毫秒）的 100% 保留、其余按 1% 或 5% 的概率保留基线。内存估算：缓冲 Trace 数约 250 × 10 × 1.2 = 3,000 条，内存约 3,000 × 10 KB ≈ 30 MB；同时 `decision_wait` 必须长于最慢的 Trace，`num_traces` 要按该公式算过再填，并配好 `memory_limiter` 防 OOM。

## 小技巧与冷知识

- **TraceID 可以直接当订单号的“影子编号”用**：出问题时让用户报订单号，后台按订单号查到 TraceID，再去 Jaeger 里搜整条链路，比让用户描述“几点几分点的哪个按钮”快得多。做法是在下单成功后把 TraceID 一起写进订单日志。
- **采样率不用一步到位**：先按 1% 跑一周，统计每天保留量和存储占用，再按实际成本反推采样率。前面算过，1% 每天约 8.24 GiB；如果存储预算每天只给 20 GiB，采样率最高可以开到约 2.4%，算式是 20 ÷ 824 ≈ 0.024。
- **给 Span 起名字要像写快递备注**：`POST /orders` 比 `handleRequest` 好一万倍；带上 HTTP 方法和路由模板，不要带真实参数值，`/orders/12345` 这种带 ID 的名字会让同类 Span 无法聚合。
- **OTel 在 2026 年 5 月从 CNCF 毕业**：和 Kubernetes、Prometheus、Envoy 同属毕业项目，意味着它已经是云原生可观测性的事实标准，选型时不用担心它突然没人维护。
- **Collector 自己也能被观测**：Collector 暴露了自身的 Metric（比如接收量、导出失败数、队列深度），把它也接进 Prometheus，就能在 Trace 丢数据时第一时间知道是 Collector 堵了还是后端挂了。
- **日志里带上 TraceID 是成本最低的关联手段**：日志框架的 MDC 里放 `trace_id` 字段，查日志时按 TraceID 一搜，同一请求的所有日志按时间排好，比在海量日志里按时间戳加关键词大海捞针快得多。

- **采样率别拍脑袋，用 Collector 自己的指标反推**：Collector 暴露了 `otelcol_receiver_accepted_spans` 指标，拿它读出真实的每秒 Span 数，再除以当前采样率，就得到采样前的真实流量；拿月存储预算除以（每条 Span 落盘字节数 × 保留天数 × 86,400），就得到能承受的每秒 Span 数上限，两者一除就是该配的采样率。量纲上记得用压缩后落盘字节数，不用线上体积，否则需求会被高估约 4 倍。

## 关联知识

- [监控告警](工程实践/运维/监控告警.md)：Metric 发现异常后靠告警触达值班人，OTel 的 Trace 与 Log 负责接力定位，告警规则与链路下钻配合才算完整闭环。
- [Prometheus与Grafana](工程实践/运维/Prometheus与Grafana.md)：OTel 产生的 Metric 经 Collector 送进 Prometheus，再由 Grafana 画图与告警，两者是“指标归 Prometheus、链路归 OTel”分工的另一半。
- [Linux性能分析](工程实践/运维/Linux性能分析.md)：Trace 定位到某台机器慢之后，用 Linux 的 CPU、内存、IO 分析工具确认是进程问题还是系统资源瓶颈。
- [JVM生产诊断工具链](开发语言/Java/JVM/JVM-生产诊断工具链.md)：Trace 定位到某个 Java 服务慢之后，用 JVM 诊断工具进一步确认是 GC、线程池还是慢方法在拖后腿。
