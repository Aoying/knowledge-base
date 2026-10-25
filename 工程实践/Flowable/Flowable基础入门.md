---
知识点: Flowable基础入门
层级: 基础
适用版本: Flowable 6.7.2
更新日期: 2026-10-25
关联:
  - Flowable
  - 工程实践知识点地图
标签:
  - Java
  - 工作流
  - 实践
---

# Flowable基础入门

> 本篇只讲入门链路：流程引擎解决什么问题、核心概念、第一个请假流程从画图到审批跑通、Spring Boot 最小集成。进阶内容见同目录三篇：事件与监听器、历史与部署、多实例与网关；全景汇总见 [工程实践/Flowable.md](工程实践/Flowable.md)。

## 流程引擎解决什么问题

没有流程引擎时，审批流长这样：`if (金额 > 10000) 走总监 else 走经理`，分支、审批人、超时、驳回全散在业务代码里。需求一变（加一级会签、换审批人、补一个条件分支），就得改代码、重新测试、重新上线，还查不到“这单当时卡在哪一步”。

流程引擎把“谁批、按什么顺序、什么条件走哪条路”从代码里抽出来，变成一份可配置的流程定义（BPMN 图）：

- **可配置**：改流程 mostly 改图，不改主业务代码。
- **可追踪**：每个实例走到哪一步、谁办的、花了多久，引擎自动留痕。
- **可审计**：审批意见、变量、历史轨迹都能查，事后可复盘。

一句话：流程引擎不是状态机的花哨写法，它是把“流转规则”从业务代码里剥离出来的运行时。

## 核心概念

| 概念 | 引擎里的对应物 | 一句话 |
|---|---|---|
| 流程定义 | `ProcessDefinition` | 一份 BPMN 图部署后得到的版本化模板，`key` 相同、版本号递增 |
| 部署 | `Deployment` | 把 BPMN 资源发布到引擎的一次操作，一次部署可含多份定义 |
| 流程实例 | `ProcessInstance` | 按某份定义跑起来的一个具体单子，如“张三的请假单” |
| 执行 | `Execution` | 实例内部的执行指针，走过一个节点就往前挪一步，并行分支会有多个执行 |
| 用户任务 | `Task` | 需要人处理的节点，如“经理审批”，进人的待办列表 |
| 流程变量 | Variable | 挂在实例上的数据，如 `days`、`applicant`，全流程可见 |
| 任务局部变量 | Local variable | 只挂在某个任务上，任务完成即销毁，结论类数据别放这里 |
| 网关 | Gateway | 分支/汇合的决策点，条件写在顺序流上 |
| 事件 | Event | 开始、结束、定时、消息等触发点 |

三个 Service 是日常打交道的入口：

- `RepositoryService`：管定义与部署（发布、查版本、挂起/激活）。
- `RuntimeService`：管实例（启动、查变量、发信号）。
- `TaskService`：管任务（查待办、签收、完成、改派）。

## BPMN 20 核心元素一张表

| 元素 | 图形 | 用途 |
|---|---|---|
| 开始事件 | 细圆圈 | 实例从这里启动，一个流程通常一个开始事件 |
| 结束事件 | 粗圆圈 | 实例到这里结束，到达即整个实例结束（除非在并行分支里） |
| 用户任务 | 圆角矩形 + 小人 | 等人审批/填写，核心节点 |
| 服务任务 | 圆角矩形 + 齿轮 | 自动执行一段逻辑，如调接口、发通知，不等人 |
| 排他网关 | 菱形 + X | 二选一/多选一，按条件只走一条分支 |
| 并行网关 | 菱形 + + | 分叉：多条分支同时走；汇合：等所有分支到齐再往下 |
| 包容网关 | 菱形 + O | 满足条件的分支都走，汇合时等走过的分支到齐 |
| 顺序流 | 带箭头实线 | 节点之间的连线，条件写在流上，如 `${days > 3}` |
| 边界事件 | 贴在任务边上的小圆圈 | 任务执行中触发，如超时 3 天未批自动转办 |
| 子流程 | 圆角矩形 + 加号 | 把一组节点打包成一个可复用单元（进阶，见多实例与网关篇） |

新手先掌握前 8 行，就能画出绝大多数审批流。

## 第一个请假流程：从画图到审批

流程：员工提交 → 天数 ≤ 3 天经理批，> 3 天总监批 → 结束。BPMN 核心片段：

```xml
<process id="leave" name="请假流程" isExecutable="true">
  <startEvent id="start"/>
  <userTask id="managerApprove" name="经理审批">
    <humanPerformer><resourceAssignmentExpression><formalExpression>manager</formalExpression>
    </resourceAssignmentExpression></humanPerformer>
  </userTask>
  <userTask id="directorApprove" name="总监审批"/>
  <exclusiveGateway id="gw" name="天数判断"/>
  <endEvent id="end"/>
  <sequenceFlow sourceRef="start" targetRef="gw"/>
  <sequenceFlow sourceRef="gw" targetRef="managerApprove">
    <conditionExpression>${days &lt;= 3}</conditionExpression>
  </sequenceFlow>
  <sequenceFlow sourceRef="gw" targetRef="directorApprove">
    <conditionExpression>${days &gt; 3}</conditionExpression>
  </sequenceFlow>
  <sequenceFlow sourceRef="managerApprove" targetRef="end"/>
  <sequenceFlow sourceRef="directorApprove" targetRef="end"/>
</process>
```

注意两点：条件表达式用 `${}` 写 EL；XML 里 `<`、`>` 要转义成 `&lt;`、`&gt;`，直接写 `<` 会导致 BPMN 解析失败。

链路四步，代码按顺序贴：

```java
// 1. 部署：资源名须以 .bpmn 或 .bpmn20.xml 结尾才会被解析为流程定义
repositoryService.createDeployment()
    .addClasspathResource("processes/leave.bpmn20.xml")
    .deploy();

// 2. 启动：按 key 起实例，变量随实例走全程
ProcessInstance inst = runtimeService.startProcessInstanceByKey(
    "leave", Map.of("applicant", "zhangsan", "days", 5));

// 3. 查待办：按办理人查，拿到任务 id 才能往下办
Task task = taskService.createTaskQuery().taskAssignee("director").singleResult();

// 4. 审批完成：结论放流程变量，别放任务局部变量（任务一销局部变量就没）
taskService.complete(task.getId(), Map.of("approved", true));
```

跑通的标志：5 天请假走到总监审批，3 天请假走到经理审批，审批完成后来查实例已结束，历史里能看到走过的节点。

## Spring Boot 集成最小配置

```xml
<!-- 只需这一个 starter，引擎、建表、事务都由它接管 -->
<dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-spring-boot-starter</artifactId>
    <version>6.7.2</version>
</dependency>
```

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/flowable?useUnicode=true&characterEncoding=utf8
    username: root
    password: xxxx
flowable:
  database-schema-update: true   # 启动时自动建表/升级表结构，生产建议先在测试库核对
  history-level: audit           # 审计够用的常用档位，见历史与部署篇
  async-executor-activate: false # 入门先关异步执行器，定时/异步任务进阶再开
```

启动后 `ACT_RE_*`（定义类）、`ACT_RU_*`（运行类）、`ACT_HI_*`（历史类）三组表自动建好。`database-schema-update: true` 会自动升级表结构，生产发布前先在测试库跑一遍、核对它要改的 DDL，别让它在生产库上悄悄改表。

## 常见误区

- **流程变量里塞大对象**：变量是序列化进数据库的，塞整个订单对象会让历史表暴涨、查询变慢。只放 id 和结论类标量，大对象放业务库、流程里只传 id。
- **审批人写死在图里**：`manager` 这种写死只适合演示。生产用候选组（`candidateGroups`）或监听器里动态算办理人，人事变动不用重新画图。
- **拿流程图当状态机用**：纯状态流转（订单状态从待支付到已发货）没有“人审批、会签、超时转办”时，用状态枚举更简单；上流程引擎的收益来自审批、会签、分支、审计这些流程语义。
- **条件分支没写默认流**：排他网关所有条件都不满足时实例会卡住。给网关留一条无条件默认流，兜底到人工处理节点。

## 与进阶篇的分工

| 已有篇目 | 管什么 | 本篇讲到哪 |
|---|---|---|
| [工程实践/Flowable.md](工程实践/Flowable.md) | 全景汇总 | 只引用，不重复 |
| [工程实践/Flowable/Flowable-事件与监听器.md](工程实践/Flowable/Flowable-事件与监听器.md) | 监听器、全局事件 | 只提“节点上可挂钩子”，不展开挂法与事务规则 |
| [工程实践/Flowable/Flowable-历史与部署.md](工程实践/Flowable/Flowable-历史与部署.md) | 历史级别、清理、版本 | 只提 history-level 常用档位，不展开清理与版本规则 |
| [工程实践/Flowable/Flowable-多实例与网关.md](工程实践/Flowable/Flowable-多实例与网关.md) | 多实例、网关高级用法 | 只列网关类型，不展开多实例与复杂汇合 |

子流程、动态条件、流程迁移、自定义表单渲染都不在本篇，归进阶篇。

## 面试与自测

1. 为什么审批流要用流程引擎，而不是在业务代码里写 if-else？
2. 流程定义、流程实例、任务三者是什么关系？一个定义能对应几个实例？
3. 排他网关、并行网关、包容网关的分支规则各是什么？
4. 流程变量和任务局部变量的生命周期有何不同？审批结论应该放哪？
5. `startProcessInstanceByKey` 默认用哪个版本的定义起实例？在途实例受新版本影响吗？

### 参考答案

1. 因为流转规则（谁批、什么顺序、什么条件走哪条路）会频繁变，写进代码就得改代码、重测、上线，且查不到单子当时卡在哪一步。流程引擎把规则抽成可配置的流程定义，改图为主，实例轨迹、审批意见、变量自动留痕，可追踪、可审计。
2. 定义是模板，实例是按定义跑起来的具体单子，任务是实例中需要人处理的节点。一个定义可对应多个实例（如每张请假单各一个实例），一个实例在某一时刻可有多个并行任务。
3. 排他网关按条件只走一条分支；并行网关分叉时所有分支同时走，汇合时等全部到齐；包容网关满足条件的分支都走，汇合时等实际走过的分支到齐。
4. 流程变量挂在实例上，全流程可见、实例结束前一直都在；任务局部变量只挂在某个任务上，任务完成即销毁。审批结论是后续节点和审计要用的数据，必须放流程变量，放局部变量会在任务完成时丢失。
5. 默认用最新版本起新实例。在途实例继续跑它启动时那版图，不受新版本影响；同 key 每次部署版本号自动 +1，旧版本定义不消失、不被覆盖。

## 关联知识

- [工程实践/Flowable.md](工程实践/Flowable.md)
- [工程实践/Flowable/Flowable-历史与部署.md](工程实践/Flowable/Flowable-历史与部署.md)
- [工程实践/Flowable/Flowable-多实例与网关.md](工程实践/Flowable/Flowable-多实例与网关.md)
- [工程实践/Flowable/Flowable-事件与监听器.md](工程实践/Flowable/Flowable-事件与监听器.md)
