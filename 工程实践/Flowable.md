---
知识点: Flowable
层级: 进阶
适用版本: Flowable 6.7.2
更新日期: 2026-10-22
关联:
  - 工程实践知识点地图
  - Java知识点地图
  - 神农-e贷-审批流
标签:
  - Java
  - 工作流
  - 实践
---

# Flowable

> 每晚 20:30 自动补充完善。贴合 Flowable 6.7.2（神农 e 贷同款）。

## 六大 Service（先记住这个）

| Service | 干什么 |
|---|---|
| RepositoryService | 部署流程定义、查定义/资源 |
| RuntimeService | 启动实例、驱动流程、发信号/消息 |
| TaskService | 人工任务：查询、签收、完成、委派、加签 |
| HistoryService | 历史实例/任务/变量查询 |
| ManagementService | 作业、死信、表元数据 |
| DynamicBpmnService | 运行时动态改流程定义（慎用） |

## Spring Boot 集成最小配置

```yaml
flowable:
  database-schema-update: true   # 自动建表/升级，生产建议 false + 手动执行 SQL
  history-level: audit            # none/activity/audit/full，按需开，full 很占表
  async-executor-activate: true   # 异步任务执行器（定时器/异步节点靠它）
```

## 常用 API 速记

```java
// 部署
repositoryService.createDeployment().addClasspathResource("process.bpmn20.xml").deploy();
// 启动（businessKey 强烈建议传业务主键，方便反查）
runtimeService.startProcessInstanceByKey("leave", businessKey, variables);
// 查我的待办
taskService.createTaskQuery().taskAssignee("ao").list();
// 完成
taskService.complete(taskId, variables);
```

## 网关选型

- **排他网关**：条件分支走一条，条件全不满足会抛异常——记得配默认流。
- **并行网关**：分支全走，汇聚等全部到齐；分支里有人工任务时会等所有人都做完。
- **包容网关**：满足条件的分支都走，汇聚等"可能到达的"分支，语义最复杂，少用。
- **事件网关**：等第一个发生的事件（定时/消息/信号），常用于超时分支。

## 子流程与多实例

- **子流程**：把一段逻辑包起来复用；**调用活动（CallActivity）** 调外部流程定义。
- **多实例**：会签/并签就用它。`isSequential=false` 并行，`completionCondition` 控制提前结束（如 `${nrOfCompletedInstances/nrOfInstances >= 0.6}` 六成通过即结束）。
- **补偿**：`cancelActivity=true` 的边界补偿事件 + 补偿处理器，用于 Saga 式回滚（已在 Gitee ob_note 的 `杂记/flowable.md` 细化）。

## 常见坑

1. **事务**：`RuntimeService` 操作默认加入当前 Spring 事务；流程里调外部 HTTP 超时又回滚不了——长耗时调用放异步任务或事件驱动。
2. **历史表膨胀**：`history-level: full` 的表涨得飞快，定期归档 `ACT_HI_*`。
3. **异步执行器没开**：定时边界事件不触发，十有八九是 `async-executor-activate` 没开或线程池满了。
4. **流程定义缓存**：`DynamicBpmnService` 改的是内存模型，重启丢；要持久改走新版本部署。
5. **节点授权**：候选人/候选组（`candidateUsers`/`candidateGroups`）和签收（claim）分开，查待办用 `taskCandidateUser` 别用错。

## 版本备注（待验证）

- Flowable 7.x 已发布，主线面向 Spring Boot 3 / Jakarta EE；6.7.x 仍是存量项目主流。升级 7.x 涉及包名 javax→jakarta，动之前先在分支验证。

## 待补充





## 疑难杂症

### 1. 流程卡在异步节点不动，也不报错
- **现象**：任务到异步节点后没下文，act_ru_job 表里有记录但一直不执行。
- **原因**：`asyncExecutorActivate` 没开，或 job 被标记为 deadletter（失败超重试次数）。
- **解决**：确认 `flowable.async-executor-activate=true`；查 `act_ru_deadletter_job` 表看失败原因；手动触发 `managementService.moveDeadLetterJobToExecutableJob` 重试。

### 2. 会签（多实例）一个人点了就结束了
- **现象**：并行会签 3 个人，1 个人审批完流程就往下走了。
- **原因**：`completionCondition` 写错了（如 `${nrOfCompletedInstances == 1}`），或压根没配。
- **解决**：全票通过配 `${nrOfCompletedInstances == nrOfInstances}`；按比例配 `${nrOfCompletedInstances/nrOfInstances >= 0.6}`；画完图务必走一遍测试。

### 3. 流程图中文乱码
- **现象**：生成的流程图图片里中文全是方框。
- **原因**：服务器（Linux/TongWeb）没有中文字体，Flowable 画图时 fallback 失败。
- **解决**：服务器装中文字体（如 `wqy-microhei`）；或在代码里指定 `activityFontName` / `labelFontName` 为已安装的字体。

## 驳回 / 回退实现方案

Flowable 原生没有"驳回"按钮，三种实现按推荐度排序：

1. **画回去**：在 exclusiveGateway 上加一条"驳回"顺序流连回上一节点，最土但最稳，历史记录完整。
2. **ChangeActivityStateBuilder**（7.x 推荐 API）：
```java
runtimeService.createChangeActivityStateBuilder()
    .processInstanceId(piId)
    .moveActivityIdTo(currentTaskKey, targetTaskKey)  // 当前节点 → 目标节点
    .changeState();
```
3. **连线条件**：网关上 `${approved == false}` 走驳回分支，变量由表单/代码设置。

- 驳回后记得把"驳回意见"写进 comment（`taskService.addComment`），不然审批人不知道为啥被打回。
- 会签里的驳回：先 `completionCondition` 放行，再用方案 2 跳回去，别在会签里画驳回线。

## 流程实例迁移（6.7 migration API）

流程定义升级 v1→v2，老实例还在 v1 上跑，两种处理：

```java
// 方案1：老实例自然结束，新实例走新定义（推荐，能等就等）
// 方案2：批量迁移
ProcessInstanceMigrationBuilder builder =
    runtimeService.createProcessInstanceMigrationBuilder()
        .migrateToProcessDefinition(newDefId)
        .addActivityMigrationMapping(
            ActivityMigrationMapping.createMappingFor("oldTask", "newTask"));
builder.migrate(processInstanceId);  // 单个迁；批量自己循环
```

- 迁移要求新老定义的 activity id 能对上，对不上的用 mapping 显式指定。
- 神农 e 贷这种金融审批流：先在测试环境迁 10 个实例验证，再全量；迁之前先备 `act_*` 表。

## 自定义节点行为：JavaDelegate vs 表达式

```java
// JavaDelegate：重逻辑
@Component("riskCheck")
public class RiskCheckDelegate implements JavaDelegate {
    public void execute(DelegateExecution ex) {
        ex.setVariable("riskScore", score(ex.getVariable("orderId")));
    }
}
```
```xml
<serviceTask id="t1" flowable:delegateExpression="${riskCheck}"/>
<!-- 轻逻辑直接表达式 -->
<serviceTask id="t2" flowable:expression="${order.amount > 10000}"/>
```

- 选型：超过 10 行、一处以上复用、要单元测试 → JavaDelegate；简单赋值/判断 → 表达式。
- delegate 丢异常整个事务回滚，想"失败继续"用 async + 边界错误事件接住。

## 神农 e 贷审批流案例

项目实战记录已移至 [项目沉淀/神农-e贷-审批流.md](项目沉淀/神农-e贷-审批流.md)，按"项目记录与通用技术知识分开"规则存放，模板里的 `[ ]` 待填实战细节。

## 基础知识补强

### 核心概念与术语

- **工作流引擎是什么**：专门负责「按预先画好的流程图推进业务」的组件。你把审批顺序、分支条件、经办人规则画成流程图，引擎负责在每一步创建任务、等人处理、按条件走下一步，并把全过程记录下来。业务代码只管业务判断，不用自己维护流程走到哪一步了。
- **BPMN 2.0**：画流程图的国际标准规范（元素长什么样、各自什么语义都有统一定义），Flowable 执行的就是它。图不是示意图，是能直接部署运行的程序，画错语义引擎就按错的跑。
- **流程定义（Process Definition）**：一张流程图的部署产物，是模板。有唯一标识 `key`（如请假流程）和版本号。改图重新部署，版本号加一，旧版本不消失。
- **流程实例（Process Instance）**：按定义跑起来的一次具体流程，如张三的一次请假申请。定义是菜谱，实例是照着菜谱做的一道菜；两者别混。
- **令牌（Token）**：理解引擎运行的核心心智模型。想象有一个令牌在流程图上移动：实例启动时令牌从开始事件出发，沿连线到达一个节点就停在那里等处理（人工任务等审批、定时器等时间）；任务完成，令牌继续往前走；遇到网关按条件选路；走到结束事件，这个令牌消失，所有令牌都消失时实例结束。并行分支就是令牌分叉成多个同时移动。
- **活动（Activity）**：流程图上能停令牌的节点统称，如用户任务、服务任务、子流程。排查日志和历史表里出现 activity 字样，说的就是节点。
- **任务（Task）**：特指等人来办的人工任务，是活动的一种。它有自己的生命周期：被创建、被签收、被完成。与之相对，服务任务是系统自动执行的，不等人。

BPMN 基础元素速览（读图最低要求）：

- **开始事件**：空心圆，流程的入口，一个图至少一个。带触发类型时表示由消息、定时等启动。
- **结束事件**：粗边圆，令牌到此消失。图里可以有多个结束事件，表示不同的结束方式。
- **用户任务**：圆角矩形加小人图标，人来办，引擎在此停住等完成。
- **服务任务**：圆角矩形加齿轮图标，系统自动执行一段代码或表达式，执行完自动往下走。
- **脚本任务**：矩形加卷轴图标，执行图里写的一段脚本，适合小段逻辑，复杂逻辑别用它。
- **顺序流**：节点之间的带箭头连线，令牌的行走路线。上面可以写条件表达式，满足才走这条线。
- **泳道（Pool/Lane）**：按部门或角色分区画图的框，只起组织视图作用，不影响引擎执行逻辑，别指望靠它做权限隔离。

### 基础语法与用法

一段最小流程图的 XML 骨架，认识每个标签对应图上的什么：

```xml
<process id="leave" name="请假流程">
  <startEvent id="start"/>
  <sequenceFlow id="f1" sourceRef="start" targetRef="apply"/>
  <userTask id="apply" name="提交申请" flowable:assignee="${applicant}"/>
  <sequenceFlow id="f2" sourceRef="apply" targetRef="approve"/>
  <userTask id="approve" name="主管审批" flowable:candidateGroups="manager"/>
  <sequenceFlow id="f3" sourceRef="approve" targetRef="end"/>
  <endEvent id="end"/>
</process>
```

- `id` 是程序引用的标识（代码和迁移都靠它），部署后轻易别改；`name` 是给人看的，随便改。
- 每个元素的 id 在同一流程定义内必须唯一，改图时顺手改 id 是很多诡异问题的来源。
- 用建模器画图时不用手写 XML，但排查部署失败必须能读 XML，建模器也是生成它。

启动与办理的最小 Java 流程，按时间顺序理解各 Service 何时出场：

```java
// 1. 部署阶段（应用启动时或发布新图时做一次）：RepositoryService 管
repositoryService.createDeployment()
    .addClasspathResource("processes/leave.bpmn20.xml")
    .deploy();

// 2. 发起：把业务单据号作为 businessKey 传进来，以后靠它双向查
Map<String, Object> vars = new HashMap<>();
vars.put("applicant", "ao");
vars.put("days", 3);
runtimeService.startProcessInstanceByKey("leave", "LEAVE-2026-001", vars);

// 3. 办理人查到自己的任务并完成，complete 时流程往下走一步
Task task = taskService.createTaskQuery()
    .taskAssignee("ao").singleResult();
taskService.complete(task.getId());
```

流程变量的基础规则：

- 变量是挂在实例上的键值数据，图里的条件表达式、办理人表达式都从这里取值。
- 启动时传的变量整个实例生命周期可读；每个任务完成时也可以再放入新变量（如审批结果）。
- 变量值要可序列化（基本类型、字符串、实现了序列化接口的对象），随手塞不可序列化的对象会在持久化时报错。
- 变量有作用域：默认挂实例全局；子流程和多实例里还有局部变量，只在本节点范围可见，同名时局部优先。

办理人相关的三个基础概念：

- **受理人（assignee）**：任务明确归某一个人，他不用抢，直接办。
- **候选人/候选组（candidate）**：任务先挂在一个池子里，组内任何人可办，但必须先签收（claim），签收后它变成签收人的个人任务，别人就办不了了。
- **签收（claim）与反签收（unclaim）**：领取和放回池子。不签收直接查个人待办是查不到候选任务的，查待办要分清查的是哪一种。

### 关键机制速览

- **定义版本如何选**：默认按 `key` 启动时用的是最新部署的版本；在途的旧实例继续按它启动时那版图跑，不会被新图影响。这就是改图不影响已发流程的原因，也是必须管理版本的原因。
- **数据表家族（ACT_ 前缀的表按中间字母分工）**：
  - `ACT_RE_*`：定义与部署资源，相对静态。
  - `ACT_RU_*`：运行中实例的当前状态，实例结束数据就删，查「现在走到哪」看它。
  - `ACT_HI_*`：历史归档，实例结束后长期保留，查「走过什么路」看它。
  - `ACT_ID_*`：Flowable 自带的用户与组（实际项目多接自己的权限体系，未必用）。
  - 排查思路：实例不见了先分清是结束了（去历史表）还是被删了，别只盯运行表。
- **事务边界**：一次 API 调用（如 complete）内的引擎操作在同一事务里，要么全成要么全回滚。任务完成时后续的自动节点会连着执行，直到遇到下一个等待点（人工任务、定时器等）才停。
- **等待点概念**：引擎只在等待点把状态持久化并释放。两个等待点之间的自动节点是一口气跑完的，中间抛异常会把这一整段回滚到上一个等待点，理解回滚范围就靠这个模型。
- **异步与定时靠执行器**：定时器、异步标记的节点不是靠主流程线程跑的，是后台作业执行器到点捞出来执行。这类节点不触发时，问题不在业务代码，在执行器和作业表。
- **历史级别决定留多少痕**：历史级别越高，历史表里记的变量和活动细节越全，审计能力越强，但表增长越快。上线前按审计需求定级别，不要默认拉满。

### 新手易错点

- **把改图当改数据**：流程图上线后发现画错了，重新部署只影响新实例，老实例照着错图继续跑。处理老实例要么让它自然结束，要么做实例迁移，不能指望重新部署自动纠正。
- **businessKey 不传**：启动时不传业务主键，事后从业务单据反查流程、从流程反查单据全靠全表扫变量，又慢又脆。启动必传，这是纪律不是可选项。
- **查待办用错查询条件**：候选任务没签收时用受理人条件查不到，误以为任务丢了。先确认任务是分配型还是候选型，再选对应的查询方式。
- **条件表达式变量没传**：网关分支靠表达式里的变量判断，变量忘了在启动或上一步完成时放入，轻则走默认分支，重则因空值报错卡住。排查看历史变量表最快。
- **在图里堆业务代码**：把大段逻辑塞进脚本任务和表达式里，图看着简单，实际没法调试、没法测、改一点全量回归。复杂逻辑放 Java 代码，图只表达流程结构。
- **拿流程变量当数据库用**：把大对象、长文本、全量列表塞进变量，每次流转都在历史表里复制一份，表膨胀极快。变量只放流转判断需要的小数据，大数据存业务表、变量里放主键。
- **结束事件理解错误**：以为走到任意一个结束事件实例就一定全结束。并行分支下其他令牌还在跑时，实例并不结束；只有全部令牌都消失才结束。画多结束事件时要配合令牌模型自查一遍。

## Flowable 实战

见 [工程实践/Flowable.md](工程实践/Flowable.md)，Java 侧的调用、驳回、网关、会签、异步 job 全在那边沉淀。

## 实战案例（项目沉淀）

- [督办系统](项目沉淀/督办系统.md)：Flowable 驱动的督办全流程
- [神农-e贷-法人版](项目沉淀/神农-e贷-法人版.md) 与 [神农-e贷-审批流](项目沉淀/神农-e贷-审批流.md)：贷款审批流实战
