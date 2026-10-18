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

项目实战记录已移至 [12-项目沉淀/神农-e贷-审批流.md](12-项目沉淀/神农-e贷-审批流.md)，按"项目记录与通用技术知识分开"规则存放，模板里的 `[ ]` 待填实战细节。
