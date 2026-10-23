---
知识点: Flowable
层级: 进阶
适用版本: Flowable 6.7.2
更新日期: 2026-10-23
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

> 本篇是 Flowable 汇总与索引：进阶主题已拆为 3 篇独立子篇，存放在 `工程实践/Flowable/` 子目录。本篇留全景导读、速查与选型，细节查子篇。

## 子篇索引

- [多实例与网关](工程实践/Flowable/Flowable-多实例与网关.md) ★：会签/或签的完成条件配方、四种网关的分路与汇聚语义、动态条件、候选人/签收等节点授权。审批流画图时最先查的一篇。
- [事件与监听器](工程实践/Flowable/Flowable-事件与监听器.md) ★：执行/任务监听器挂钩子、全局事件监听做审计告警、信号/消息/trigger 从外部唤醒流程、异步与死信、「汇总节点」的组合实现。
- [历史与部署](工程实践/Flowable/Flowable-历史与部署.md)：历史级别与清理、部署与版本规则、回退的三种现实做法（旧图重部署、节点回退、实例迁移）与驳回实现。

## 全景导读

Flowable 是执行 BPMN 2.0 流程图的引擎：你把审批顺序、分支条件、经办人规则画成图，引擎负责在每一步创建任务、等人处理、按条件推进，并全程留痕。理解它只需要一个心智模型——**令牌**：实例启动时令牌从开始事件出发，到达节点就停下等处理（人工任务等人、定时器等时间），处理完继续走；遇到网关按条件分路，并行时令牌分叉成多个，全部消失时实例结束。

三个基本对象别混：**流程定义**是部署后的模板（有 key 和版本号，改图重部署只加版本、不影响在途实例）；**流程实例**是按定义跑起来的一次业务（如一笔贷款申请）；**任务**特指等人办的人工任务，是活动的一种，服务任务则由系统自动执行。

数据表按 `ACT_` 前缀分工：`ACT_RE_*` 存定义与部署（静态）、`ACT_RU_*` 存运行中状态（实例结束即删，查「现在走到哪」）、`ACT_HI_*` 存历史归档（查「走过什么路」，细则见历史与部署篇）、`ACT_ID_*` 是自带用户组（项目多接自己的权限体系）。事务边界是一次 API 调用：调用内的引擎操作同事务提交，自动节点会一口气跑到下一个等待点（人工任务、定时器等）才停，中间抛异常整段回滚到上一个等待点——理解回滚范围就靠这个模型。

## 六大 Service（速查）

| Service | 干什么 |
|---|---|
| RepositoryService | 部署流程定义、查定义/资源 |
| RuntimeService | 启动实例、驱动流程、发信号/消息 |
| TaskService | 人工任务：查询、签收、完成、委派、加签 |
| HistoryService | 历史实例/任务/变量查询 |
| ManagementService | 作业、死信、表元数据 |
| DynamicBpmnService | 运行时动态改流程定义（只改内存模型、重启丢，慎用） |

## 最小配置与常用 API

```yaml
flowable:
  database-schema-update: true   # 自动建表/升级，生产建议 false + 手动执行 SQL
  history-level: audit           # none/activity/audit/full，按需开，full 很占表
  async-executor-activate: true  # 异步任务执行器（定时器/异步节点靠它）
```

```java
// 部署
repositoryService.createDeployment().addClasspathResource("process.bpmn20.xml").deploy();
// 启动（businessKey 强烈建议传业务主键，方便反查）
runtimeService.startProcessInstanceByKey("leave", businessKey, variables);
// 查我的待办（候选任务要用 taskCandidateUser，别用错，见多实例与网关篇）
taskService.createTaskQuery().taskAssignee("ao").list();
// 完成
taskService.complete(taskId, variables);
```

自定义节点行为二选一：逻辑超过 10 行、要复用或单测 → `JavaDelegate`（`flowable:delegateExpression="${bean}"` 走 Spring Bean）；简单赋值/判断 → `flowable:expression` 表达式或脚本任务。delegate 抛异常整个事务回滚，想「失败继续」把节点标异步、用边界错误事件接住。

## 版本与选型要点

- 本库口径是 **Flowable 6.7.2**（Spring Boot 2.7 存量项目的事实标配）：API 与表结构稳定，`ChangeActivityStateBuilder`、实例迁移等 6.7 的新 API 都可用。
- Flowable 7.x 已发布，主线面向 Spring Boot 3 / Jakarta EE，升级涉及包名 javax→jakarta 与依赖大换血，存量项目动之前先在分支验证（待验证：7.x 的具体差异以官方迁移指南为准）。
- 选型一句话：Activiti 是同源前身、社区节奏慢；Camunda 7 商业化后社区版受限、Camunda 8 是另一套架构。已在 Spring Boot 2 存量栈里，Flowable 6.7.2 是成本最低的选择。

## 常见坑总览（细节在子篇）

- 异步执行器没开，定时边界事件永不触发 → [事件与监听器](工程实践/Flowable/Flowable-事件与监听器.md)
- 会签一个人点完就结束，完成条件写错 → [多实例与网关](工程实践/Flowable/Flowable-多实例与网关.md)
- 候选任务用 assignee 条件查不到，误报「任务丢了」 → [多实例与网关](工程实践/Flowable/Flowable-多实例与网关.md)
- `history-level: full` 历史表膨胀 → [历史与部署](工程实践/Flowable/Flowable-历史与部署.md)
- 流程图中文乱码：服务器缺中文字体，装字体（如 `wqy-microhei`）或代码指定 `activityFontName` / `labelFontName`。
- 流程里同步调外部 HTTP：超时且回滚不了，长耗时调用放异步任务或事件驱动。

## 实战案例（项目沉淀）

- [神农-e贷-审批流](项目沉淀/神农-e贷-审批流.md)：会签多数决、边界定时器超时转办、补偿回滚、回任意节点的真实配置与坑。
- [神农-e贷-法人版](项目沉淀/神农-e贷-法人版.md)、[督办系统](项目沉淀/督办系统.md)：Flowable 在项目里的整体用法。
