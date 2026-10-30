---
知识点: Flowable-事件与监听器
层级: 进阶
适用版本: Flowable 6.7.2
更新日期: 2026-10-31
关联:
  - Flowable
  - 工程实践知识点地图
标签:
  - Java
  - 工作流
  - 原理
  - 实践
---

# Flowable-事件与监听器

> 汇总与全景见 [工程实践/Flowable.md](工程实践/Flowable.md)。监听器是往流程节点上挂钩子的机制：不进主流程代码，在节点生命周期的关键点插入逻辑。

## 两类监听器

| 类型 | 挂载点 | 典型事件 | 入口接口 |
|---|---|---|---|
| 执行监听器 ExecutionListener | 流程、活动、连线 | `start`、`end`；连线上是 `take` | `ExecutionListener.notify(DelegateExecution)` |
| 任务监听器 TaskListener | 用户任务 | `create`、`assignment`、`complete`、`delete`；`all` 表示全部 | `TaskListener.notify(DelegateTask)` |

三种挂法，按推荐度：

```xml
<!-- 1. delegateExpression：Spring Bean，可注入、可测，首选 -->
<userTask id="approve" name="审批">
  <extensionElements>
    <flowable:taskListener event="create" delegateExpression="${auditTaskListener}"/>
  </extensionElements>
</userTask>
<!-- 2. class：全类名，引擎自己 new，不走 Spring 注入 -->
<!-- 3. expression：${bean.method(execution)}，轻量逻辑用 -->
```

```java
@Component("auditTaskListener")
public class AuditTaskListener implements TaskListener {
    public void notify(DelegateTask task) {
        // create 时任务还没落库定型，适合补默认值、改办理人
        task.setVariableLocal("createTime", new Date());
    }
}
```

- 执行顺序：同一事件挂多个监听器时按 XML 声明的先后执行；任务监听器的 `assignment` 在 `create` 之后触发，签收/改派时也会再触发，别在里面写只该跑一次的逻辑。
- 监听器与主流程**同一个事务**：抛异常则当前这次 API 调用整体回滚。想「记日志失败别影响审批」就在监听器内部自己 catch。
- `complete` 事件里结论类数据要写流程变量：任务马上销毁，写任务局部变量会随任务一起没。

## 全局事件监听：盯整个引擎

节点级监听器要逐个图挂；全局视角（审计、监控、告警）用事件监听器，在引擎配置里统一注册：

```java
// 注册：给 Spring 引擎配置加一个 EngineConfigurationConfigurer，把监听器塞进去
cfg -> cfg.setEventListeners(List.of(auditEventListener))

@Component
public class AuditEventListener extends AbstractFlowableEngineEventListener {
    @Override
    protected void processCompleted(FlowableEvent event) { /* 流程结束落审计 */ }
    @Override
    protected void jobExecutionFailure(FlowableEvent event) { /* 作业失败告警 */ }
}
```

- 事件类型在 `FlowableEngineEventType` 里：任务创建/完成、流程启动/结束/取消、作业执行失败等，覆盖绝大多数观测需求。
- `isFailOnException()` 默认 false：全局监听器抛异常默认不打断主流程（与节点监听器相反）。要发外部通知（MQ、HTTP）别在事务里直接发——主流程回滚了通知已发出去是经典不一致；可结合事务生命周期事件等提交后再触发。

## 信号、消息与 trigger：从外部唤醒流程

流程停在等待点（接收任务、事件子流程、中间捕获事件）时，靠三类外部刺激推进：

| 方式 | 作用域 | API | 语义 |
|---|---|---|---|
| trigger | 单个执行 | `runtimeService.trigger(executionId)` | 推过接收任务，无数据语义 |
| 消息 | 指定实例 | `runtimeService.messageEventReceived(name, executionId, vars)` | 与实例绑定，先查到在等消息的 executionId |
| 信号 | 全局广播 | `runtimeService.signalEventReceived(name, vars)` | 所有在等该信号的实例一起被唤醒 |

- 接收任务没有代码、没有办理人，是「等外部系统回调」的占位符，回调到了就 `trigger`；查它用 `runtimeService.createExecutionQuery().activityId("waitId")`。
- 信号是广播：同名信号被多个定义订阅时一次全醒，命名带业务域前缀防误触发；只唤醒单个实例用带 executionId 的重载。
- 消息可**启动**流程（消息启动事件）：`startProcessInstanceByMessage(name, businessKey, vars)`，适合「上游来单才建流程」。

## 异步、定时与死信

- 节点标 `flowable:async="true"` 后落一条 job 到 `ACT_RU_JOB`，由异步执行器捞起执行——长耗时调用（外部 HTTP、批量计算）都走异步，别堵办理人的请求线程。
- 定时器同样靠执行器到点触发；执行器没开（`async-executor-activate`）或线程池打满，表现就是「时间到了流程不动」。
- job 抛异常自动重试（默认 3 次），耗尽进 `ACT_RU_DEADLETTER_JOB`。修好后用 `managementService.moveDeadLetterJobToExecutableJob(jobId)` 放回重试，别直接改表。

## 「流程汇总节点」怎么做

Flowable 没有原生的汇总节点元素，「各分支结论汇总到一处再决策」靠组合实现：

1. **网关汇聚 + 执行监听器**：汇聚点后接服务任务，或在汇聚活动的 `start` 执行监听器里读各分支写回的结论变量，汇总后写成流程变量供后续网关分流。各分支结论写进**带分支标识的变量**，别共用一个变量名互相覆盖。
2. **多实例子流程收口**：把「每人审一遍」放进多实例子流程，各实例完成时在任务监听器里往集合型流程变量 append 结论，子流程天然就是汇合点，出来后统一判断。会签比例判断也是这条路。
3. 纪律：汇总逻辑只读变量、只算一次、结果落变量；不要在监听器里再去 complete 别的任务，容易把令牌模型搅乱。

## 常见坑

1. **任务监听器 class 挂法注入失败**：`class` 挂法由引擎反射 new，`@Autowired` 的字段全是 null；要 Spring 注入必须用 `delegateExpression`。
2. **create 监听器里查任务查不到**：`create` 触发时任务还没持久化完，在监听器里用 `taskService.createTaskQuery()` 按 id 查会查不到，直接用入参 `DelegateTask` 本身。
3. **信号误广播**：随手发信号把一堆同名等待实例全唤醒；信号名、消息名按业务域命名并纳入发布检查。
4. **事件监听器里做重活**：全局监听器在引擎线程同步执行，查慢 SQL、调远程接口会拖慢每次任务办理；重活投递到自己的线程池或队列，监听器里只留轻量投递。
5. **异步节点异常被吞**：异步 job 的异常不回到办理人请求里，接口返回成功、流程却死在死信表。上线清单里必须有死信表监控项。

## 小技巧与冷知识

【小技巧】流程级审计不用逐个节点挂监听器：ExecutionListener 可以直接挂在 `<process>` 元素的 `start` / `end` 上，流程启动和结束时各触发一次，在里面记开始时间、结束时间就能算出整条流程的耗时。节点级监听器只在需要定位“卡在哪一步”时再挂，流程级先搭好审计骨架。

【小技巧】assignment 事件想只通知一次，要自己打标记：`assignment` 在首次分配、签收、改派时都会触发，监听器里不能默认“只跑一次”。做法是在首次触发时用 `task.setVariableLocal("assignedNotified", true)` 打标记，之后再触发先看标记、有标记就跳过；标记用任务局部变量，别用流程变量，免得同流程其他任务互相干扰。

【小技巧】delete 事件是清理现场的钩子，不是业务收尾的钩子：任务被删除（流程实例被取消/终止、任务被 API 删除）时触发 `delete`，适合释放外部占用、删临时文件、注销回调。别把“审批通过”这类业务逻辑放进 delete 里——任务正常走完触发的是 `complete`，不是 `delete`，两条路不要混用。

【冷知识】`take` 事件只能挂在连线上：ExecutionListener 的 `take` 只在 sequence flow 上有效，令牌经过连线时触发；把它挂在活动上不会触发。想在令牌离开某个节点时插逻辑，要么挂连线的 `take`，要么挂活动的 `end`，别在活动上等 `take`。

【小技巧】死信 job 放回重试之前，先看异常堆栈再动手：`managementService.createDeadLetterJobQuery().jobId(jobId).singleResult().getExceptionStacktrace()` 能拿到 job 最后一次失败的堆栈。根因没修就放回，放回后会立刻又失败进死信；先确认堆栈里的根因已修好，再调 `moveDeadLetterJobToExecutableJob`。

【小技巧】监听器不用启动引擎就能单测：ExecutionListener 和 TaskListener 只是普通接口，直接 new 一个监听器、传 mock 的 DelegateExecution / DelegateTask 调 `notify` 就能测逻辑。想测得顺，监听器里的 Spring 依赖用构造器注入，别只留字段注入——字段注入在单测里还得靠反射塞值。

【冷知识】信号启动事件会同时启动多个流程定义：同名信号被多个流程定义订阅时，一次 `signalEventReceived` 会给每个订阅的流程定义各建一个实例；消息启动事件是 `startProcessInstanceByMessage` 指定某一个流程定义。信号名不带流程域前缀时，很容易一次信号多建几个本不该建的实例。

【小技巧】并行多实例收口时，别让各实例往同一个集合变量里 append 结论：两个实例同时完成、同时读写同一个集合变量，可能触发乐观锁冲突，其中一个失败重试。更稳的做法是每个实例写自己编号的变量（如 `conclusion_1`、`conclusion_2`），等子流程收口、令牌回到单线之后再统一汇总成集合。

## 面试与自测

1. 执行监听器和任务监听器的挂载点、典型事件、入口接口各是什么？
2. 节点监听器抛异常和全局事件监听器抛异常，对主流程的影响有什么不同？
3. 异步 job 抛异常后会怎样？耗尽重试进了死信表，正确的恢复办法是什么？

### 参考答案

1. 执行监听器挂在流程、活动、连线上，典型事件是 `start`、`end`，连线上是 `take`，入口是 `ExecutionListener.notify(DelegateExecution)`；任务监听器挂在用户任务上，典型事件是 `create`、`assignment`、`complete`、`delete`（`all` 表示全部），入口是 `TaskListener.notify(DelegateTask)`。
2. 节点监听器与主流程在同一个事务里，抛异常则当前这次 API 调用整体回滚，想“记日志失败别影响审批”要在监听器内部自己 catch；全局事件监听器的 `isFailOnException()` 默认是 false，抛异常默认不打断主流程，两者正好相反。
3. job 抛异常会自动重试，默认 3 次，耗尽后进入 `ACT_RU_DEADLETTER_JOB` 死信表。修好后用 `managementService.moveDeadLetterJobToExecutableJob(jobId)` 把它放回重试，别直接改表；另外异步 job 的异常不会回到办理人请求里，接口返回成功、流程却可能死在死信表，所以上线清单里必须有死信表监控项。

## 关联

- 多实例与网关（分支结论从哪来）：[工程实践/Flowable/Flowable-多实例与网关.md](工程实践/Flowable/Flowable-多实例与网关.md)
- 历史与部署（死信、作业的历史侧）：[工程实践/Flowable/Flowable-历史与部署.md](工程实践/Flowable/Flowable-历史与部署.md)
- 案例（边界定时器转办、自动节点异常转 BpmnError）：[项目沉淀/神农-e贷-审批流.md](项目沉淀/神农-e贷-审批流.md)
