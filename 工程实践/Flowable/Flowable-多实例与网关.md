---
知识点: Flowable-多实例与网关
层级: 进阶
适用版本: Flowable 6.7.2
更新日期: 2026-10-31
关联:
  - Flowable
  - 工程实践知识点地图
标签:
  - Java
  - 工作流
  - 实践
---

# Flowable-多实例与网关

> 汇总与全景见 [工程实践/Flowable.md](工程实践/Flowable.md)。本篇只讲两件事：一个节点怎么变成「多人办」（多实例），令牌怎么分路与汇合（网关）。

## 多实例：一个活动开 N 个分身

多实例（Multi-Instance）是把同一个活动按集合复制出多个实例并行或顺序执行的机制，会签、或签、批量处理都靠它。配置挂在活动（用户任务、服务任务、子流程、调用活动均可）上：

```xml
<userTask id="countersign" name="会签审批">
  <multiInstanceLoopCharacteristics isSequential="false"
      flowable:collection="${approverList}" flowable:elementVariable="approver">
    <completionCondition>${nrOfCompletedInstances == nrOfInstances}</completionCondition>
  </multiInstanceLoopCharacteristics>
</userTask>
```

每个实例的办理人直接在 userTask 上写 `flowable:assignee="${approver}"`，引擎为每个实例求值一次。

核心变量（引擎在多实例作用域内自动维护，只读）：

| 变量 | 含义 |
|---|---|
| `nrOfInstances` | 实例总数（并行时启动即定，顺序时随推进变化） |
| `nrOfActiveInstances` | 当前还活着的实例数 |
| `nrOfCompletedInstances` | 已完成的实例数 |
| `loopCounter` | 当前实例在集合中的下标 |

- `isSequential=false` 并行：全部实例同时创建，适合会签；`true` 顺序：一个办完才建下一个，适合逐级过人。
- 实例来源二选一：`flowable:collection`（表达式求出集合）+ `elementVariable`（迭代元素放进的局部变量），或固定次数 `loopCardinality`。
- `completionCondition` 是 UEL 表达式，**每个实例完成时求值一次**，为 true 就销毁其余活实例、令牌离开节点；不写则等全部完成。

### 会签与或签的配方

- **会签（全票）**：并行多实例 + `${nrOfCompletedInstances == nrOfInstances}`，或干脆不写完成条件。
- **会签（比例通过）**：`${nrOfCompletedInstances/nrOfInstances >= 0.6}`，六成完成即结束，剩余待办被销毁。注意统计的是「完成数」不是「同意数」——按同意票数判断要在完成条件里对结论变量另行统计，或改用监听器汇总（见 [工程实践/Flowable/Flowable-事件与监听器.md](工程实践/Flowable/Flowable-事件与监听器.md)）。
- **或签（一人通过即过）**：并行多实例 + `${nrOfCompletedInstances == 1}`。一人办完其余实例全部销毁，其他人的待办随之消失——这是设计如此，不是 bug。
- **驳回即否决** 没有原生开关：任务监听器发现驳回结论时置标志变量，完成条件带上该标志提前结束，后续网关按标志走驳回分支。

### 动态条件

- 网关与完成条件的表达式在**令牌到达时求值**，取当时的流程变量。「金额超过 X 才走总监节点」这类动态规则，本质是上游把变量准备好、条件写 `${amount > 10000}`。
- 变量没传时对 null 求值可能抛异常，条件里先判空：`${amount != null && amount > 10000}`。
- 运行时改已部署定义的条件只能走 `DynamicBpmnService` 改内存模型（不落库、重启丢）；正经做法是改图发新版本，见 [工程实践/Flowable/Flowable-历史与部署.md](工程实践/Flowable/Flowable-历史与部署.md)。

## 网关：分路与汇聚

| 网关 | 分路语义 | 汇聚语义 |
|---|---|---|
| 排他（Exclusive） | 按顺序求值，走第一条为 true 的 | 无汇聚语义，令牌各自穿过 |
| 并行（Parallel） | 无条件全部分支各出一个令牌 | 等所有入边令牌到齐才放行 |
| 包容（Inclusive） | 所有为 true 的分支都走 | 等「所有可能到达的」分支到齐 |
| 事件（Event） | 不分路，后接中间捕获事件，谁先触发走谁 | — |

- 排他网关条件全不满足会抛异常，务必配 `default` 顺序流兜底；条件按定义顺序求值，第一条命中就停。
- 并行网关分叉与汇聚是同一个元素、必须配对；汇聚处只要有一条分支的令牌永远到不了（如分支走了排他岔路），整个实例就死等。
- 包容网关汇聚最容易踩坑：引擎要推算「哪些分支的令牌还可能到达」，图里有循环、嵌套包容网关时推算可能不符合直觉，表现为提前放行或永远等。纪律：只在结构简单、分支无循环时用，复杂汇合改用多实例子流程收口。
- 事件网关后接中间捕获事件（定时、消息、信号），用于「等结果或等超时，谁先来走谁」的竞速场景，落败分支自动取消。

## 节点授权：任务到底归谁

多实例和网关决定「走几条路」，授权决定「每条路上的人工任务归谁办」，三个概念别混：

- **受理人 assignee**：任务直接归属某人，不需要签收，查待办用 `taskAssignee`。
- **候选人/候选组 candidateUsers / candidateGroups**：任务进池子，组内人先 `claim` 签收才变成自己的；查池子用 `taskCandidateUser` / `taskCandidateGroup`，**用 assignee 条件查候选任务永远查不到**，这是最高频的「任务丢了」假象。
- **签收与反签收**：`taskService.claim(taskId, userId)`、`unclaim(taskId)` 放回池子；已签收的任务别人再 claim 会抛异常，转办用 `setAssignee`，委派用 `delegateTask`（委派人办完任务回到原受理人，与转办不同）。

```java
// 候选任务：先查池子，再签收，再办
Task task = taskService.createTaskQuery()
        .taskCandidateUser("ao")          // 或 .taskCandidateGroup("deptManager")
        .singleResult();
taskService.claim(task.getId(), "ao");
taskService.complete(task.getId(), variables);
```

授权信息落在 `ACT_RU_IDENTITYLINK` 表；候选组 id 用自己权限体系的标识即可，Flowable 不校验组是否真实存在。

## 常见坑

1. **完成条件统计口径写错**：用 `nrOfActiveInstances` 判断结束——实例完成时 active 立减，条件可能在第一个人办完时就误触。结束判断只用 completed 与 instances 两个数。
2. **并行汇聚死等**：某分支里套了排他分支，令牌没回到汇聚点。画图后逐条分支数令牌：出去几个，必须回来几个。
3. **多实例结论互相覆盖**：任务完成回写变量容易把结论写到实例全局，后完成者覆盖先完成者。每实例结论用带办理人的变量名存，或汇总进集合变量。
4. **集合在运行中被改**：并行多实例的实例数在创建时就按当时的 collection 定死，运行中往流程变量里再加人不会多出实例。临时加签要另想办法（流程外加并行任务、或改图），别指望改变量。

## 关联

- 事件与监听器（完成条件之外的汇总手段）：[工程实践/Flowable/Flowable-事件与监听器.md](工程实践/Flowable/Flowable-事件与监听器.md)
- 案例（会签比例通过、边界定时器转办）：[项目沉淀/神农-e贷-审批流.md](项目沉淀/神农-e贷-审批流.md)

## 小技巧与冷知识

【小技巧】elementVariable 是局部变量，出了多实例节点就读不到：`${approver}` 只在各自的实例作用域里有效，多实例节点一结束就消失。后续网关想按“某个人批了什么”判断，不能直接读 elementVariable，要在每个实例完成时把结论写到流程变量（或带办理人后缀的变量）里；局部变量当临时工，流程变量才是跨节点的传话筒。

【冷知识】网关出边上写的条件不是每种网关都认：排他网关的 default 顺序流即使写了条件表达式也会被忽略，条件全不满足时它直接兜底；并行网关分叉时各出边上的条件同样会被忽略，所有分支无条件各出一个令牌。所以给 default 流写条件、给并行分支写条件都是白写，条件只在排他、包容网关的普通出边上才生效。

【小技巧】分支数要到运行时才知道时，别画并行网关：并行网关的分支数在画图时就定死了，几个人就得画几条分支。人数来自集合、运行时才定的场景，直接用多实例子流程——一个子流程复制 N 份，每份走同一条路，分支数由集合大小决定，图上永远只有一条线。

【冷知识】多实例活动本身可以挂边界定时器，做的是整体超时：给会签节点挂一个边界定时器，超时后整个会签活动被打断、走超时分支；想给每个实例各算各的超时，定时器要挂在实例内部的环节上。两种挂法的超时对象完全不同，画图前先想清楚要掐的是“整个会签别拖过三天”，还是“每个人别拖过三天”。

【小技巧】已签收的任务用 candidate 条件查不到：claim 之后任务就离开了池子，`taskCandidateUser` 再也查不到它，查“谁正在办”要改用 `taskAssignee`。反过来，`taskCandidateUser` 只适合查“池子里还剩谁没领”；已签收、待签收是两拨任务，查哪拨用哪路条件，混用就是原文说的“任务丢了”假象的另一面。

【小技巧】或签、比例通过时剩余实例是被“销毁”的：完成条件一满足，引擎直接销毁其余活实例，它们的任务不会走 `complete`，而是走 `delete` 事件。清理逻辑（释放占用、删临时文件）要放在 delete 监听器里，别放在 complete 里——被销毁的实例根本不会触发 complete，写在 complete 里的清理对它们永远跑不到。

【冷知识】delegateTask 委派后，`ACT_RU_TASK` 的 OWNER_ 列留的是原受理人，ASSIGNEE_ 列变成被委派人；被委派人办完 resolve 之后，assignee 会回到原受理人。查任务表时同时看这两列，就能还原“谁委派给谁、现在在谁手里”的整条链；只看 assignee 一列，会以为任务本来就属于被委派人。

【小技巧】顺序多实例里 `loopCounter` 就是“第几个人”，可以拿来做轮次规则：比如第一个人复核、其余人审批，条件里按 `loopCounter` 分流。但别拿它当“已完成人数”用——结束判断只认 `nrOfCompletedInstances` 与 `nrOfInstances`，loopCounter 是当前实例的下标，不是完成计数，混用会在第一个人办完时误触结束条件。

## 面试与自测

1. 会签比例通过的配方统计的是什么数？为什么不能直接用“同意数”写完成条件？
2. 包容网关汇聚为什么最容易踩坑？原文给的纪律是什么？
3. 候选任务为什么用 assignee 条件查不到？正确的查、签收、办结流程是什么？转办和委派有什么区别？

### 参考答案

1. 配方 `${nrOfCompletedInstances/nrOfInstances >= 0.6}` 统计的是“完成数”，不是“同意数”——六成完成即结束，剩余待办被销毁。按同意票数判断要在完成条件里对结论变量另行统计，或改用监听器汇总（见事件与监听器篇）。
2. 因为引擎要推算“哪些分支的令牌还可能到达”，图里有循环、嵌套包容网关时推算可能不符合直觉，表现为提前放行或永远等。纪律是：只在结构简单、分支无循环时用包容网关，复杂汇合改用多实例子流程收口。
3. 因为候选任务是进池子的，组内人先 `claim` 签收才变成自己的；查池子要用 `taskCandidateUser` / `taskCandidateGroup`，用 assignee 条件查候选任务永远查不到，这是最高频的“任务丢了”假象。正确流程是：用 `taskCandidateUser` 查池子 → `claim` 签收 → `complete` 办结。转办用 `setAssignee`，委派用 `delegateTask`，委派人办完任务回到原受理人，与转办不同。
