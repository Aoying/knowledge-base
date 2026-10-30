---
知识点: Flowable-历史与部署
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

# Flowable-历史与部署

> 汇总与全景见 [工程实践/Flowable.md](工程实践/Flowable.md)。本篇管流程的「身后事」：历史数据留多少、怎么清，流程定义怎么发版、出问题怎么退。

## 历史级别：留多少痕是上线前定的

`flowable.history-level` 四档，级别决定 `ACT_HI_*` 表里写什么：

| 级别 | 记录内容 | 代价 |
|---|---|---|
| `none` | 什么都不记 | 无审计能力，生产别用 |
| `activity` | 历史流程实例 + 活动实例 | 查得到走过哪些节点，查不到变量 |
| `audit` | 再加历史任务、变量、表单等明细 | 常用档位，审计够用 |
| `full` | 再加变量每次变更的明细（`ACT_HI_DETAIL`）等 | 表涨得最快，金融审计明确要求再开 |

历史表家族：`ACT_HI_PROCINST`（实例）、`ACT_HI_TASKINST`（任务）、`ACT_HI_ACTINST`（活动）、`ACT_HI_VARINST`（变量当前值）、`ACT_HI_DETAIL`（变量变更流水，full 才全）。查询统一走 `HistoryService` 的 `createHistoricXxxQuery()`，注意历史查询默认只查已结束的要显式区分：查在途实例的历史任务要加 `.unfinished()`。

## 历史清理

- 历史只能增不能改口径：实例在运行表里结束后整份搬进历史表，长期不清，`ACT_HI_*` 就是全库最大的表。
- 单个清：`historyService.deleteHistoricProcessInstance(processInstanceId)`，会连带删该实例的历史任务/活动/变量；批量清按结束时间查出 id 分批删，别一条 SQL 直删大表（锁表 + 主从延迟）。
- 6.7 起引擎提供历史清理作业（按保留时长自动删过期历史），配置项与行为以官方文档为准，开启前先在测试环境核对它对 `ACT_HI_*` 各表的处理范围，生产上更稳妥的做法仍是：先归档到历史库（或导出），再按保留期分批删，并避开业务高峰。
- 清理前先想清楚审计保留期：审批流的审计要求往往以年计，清理策略要和合规口径对齐，不是纯技术决定。

## 部署与版本

```java
Deployment deployment = repositoryService.createDeployment()
    .name("法人贷款审批流")
    .addClasspathResource("processes/corp-loan.bpmn20.xml")  // 资源名须以 .bpmn 或 .bpmn20.xml 结尾才会被解析为流程定义
    .deploy();
// 查某 key 的最新版本定义
ProcessDefinition def = repositoryService.createProcessDefinitionQuery()
    .processDefinitionKey("corpLoan").latestVersion().singleResult();
```

版本规则只有三条，记牢省一半事故：

1. 同一个 `key` 每次部署版本号自动 +1，旧版本定义**不消失、不被覆盖**。
2. `startProcessInstanceByKey` 默认用**最新版本**起新实例；在途实例继续跑它启动时那版图，互不影响。
3. 定义可挂起/激活：`suspendProcessDefinitionById` 后该版本不能再起新实例（在途实例默认继续跑，除非挂起时选了连实例一起挂起）；`activateProcessDefinitionById` 恢复。

删除部署 `repositoryService.deleteDeployment(deploymentId, cascade)`：`cascade=false` 时有实例在跑删不掉；`true` 连运行/历史数据一起删，生产禁用随手 cascade。同一份 BPMN 重复部署只会多产生一个版本而不报错，发布流程里建议做资源校验，避免「改了个空格多一版」。

`database-schema-update` 生产纪律：设 `false`，引擎表升级走官方 SQL 脚本人工执行；开发环境用 `true` 图省事即可。

## 回退与 rollback 的现实做法

Flowable 没有「定义版本一键 rollback」按钮，所谓回退是三件事的组合，按场景选：

1. **新实例回退到旧图**：不需要真回滚——把旧版本的 BPMN **重新部署一次**，它会以最新版本号成为默认启动版本（内容是旧图、版本号是新的）。比直接改库里版本号安全得多。要立刻止血也可以先挂起出问题的新版本定义，让新单子暂时起不来（或起旧版，若旧版仍是次新且未挂起需配合启动时指定定义 id）。
2. **在途实例退回某个节点（驳回/回退办理）**：这不是版本问题，是令牌位置问题，用 `ChangeActivityStateBuilder`（6.7 起的正式 API）：

```java
runtimeService.createChangeActivityStateBuilder()
    .processInstanceId(piId)
    .moveActivityIdTo("currentTaskKey", "targetTaskKey")  // 当前节点 → 目标节点
    .changeState();
```

   审批驳回三方案的推荐顺序：图上画驳回线（最稳、历史完整）＞ ChangeActivityStateBuilder 动态跳转（灵活、适合回任意节点）＞ 网关条件分支。跳转后记得用 `taskService.addComment` 写驳回意见，并清理目标节点可能残留的旧任务数据防幽灵待办；会签节点内的驳回不要在多实例里画线，先让完成条件放行再整体跳转（机制见 [工程实践/Flowable/Flowable-多实例与网关.md](工程实践/Flowable/Flowable-多实例与网关.md)）。
3. **在途实例迁到新图**：定义改动较大、老实例又等不起自然结束时用迁移 API：

```java
runtimeService.createProcessInstanceMigrationBuilder()
    .migrateToProcessDefinition(newDefId)
    .addActivityMigrationMapping(ActivityMigrationMapping.createMappingFor("oldTask", "newTask"))
    .migrate(processInstanceId);
```

   迁移要求新老 activity id 能对应，对不上的显式 mapping；迁之前备份 `ACT_*` 表、先小批量验证再全量。经验口径：能等老实例自然结束就别迁，迁移只用在图改错了必须纠偏的场景。

## 常见坑

1. **重复部署刷版本号**：应用每次启动都无条件 deploy 一遍，版本号无意义暴涨，还掩盖真正的变更。部署前先按 key + 资源校验和判断是否已部署同内容版本。
2. **删部署连历史一起没了**：`cascade=true` 把审计历史删干净，事后对账无据。删定义前先确认没有在途与待保留历史。
3. **history-level 中途调低**：从 full 改 audit 只对新数据生效，老明细还在；调高同理不补历史。改级别按「即时生效于新实例」理解，别指望它回溯。
4. **图上线靠改库救火**：直接改 `ACT_RE_*` 里的定义或版本号，绕过部署与缓存，节点间状态不一致，出问题没法查。任何图变更只走部署 API。

## 小技巧与冷知识

【小技巧】历史清理作业默认是关闭的，开启前先把保留期对齐审计口径：默认每天凌晨 1 点清理 365 天前结束的实例（`flowable.enable-history-cleaning=true`、`flowable.history-cleaning-after=365d`），但审批流的审计保留期往往以年计，365 天未必够用。别照抄默认值就上线，先问合规要保留年限，再定清理周期，清理策略是合规决定不是纯技术决定。

【小技巧】查历史任务要分清 `.finished()` 和 `.unfinished()`：历史查询默认不区分在途还是已结束，只统计“已办任务平均耗时”时要加 `.finished()`，只查“在途待办”时加 `.unfinished()`。不加区分就把两种任务混在一起算，平均耗时会被大量未结束的任务拉偏，待办清单里也会混进已办完的任务。

【小技巧】删部署前先用运行查询数一遍在途实例再动手：`runtimeService.createProcessInstanceQuery().processDefinitionId(defId).count()`，count 为 0 再删。`cascade=false` 时有实例在跑会直接删不掉，白白报一次错；count 先查一遍，删之前就知道这次能不能删、要不要先等在途实例跑完或迁移。

【小技巧】重新部署旧图回退时，版本号不会变回去，只会继续 +1：旧图重新部署后成为最新版本，但它的版本号是新号不是旧号。核对回退是否生效要看定义 id 或 deployment id，别在监控里盯着“版本号变小了没有”判断——按这个标准看，回退永远像没生效。

【冷知识】版本号是按 key 递增的，不是全库递增：每个 key 的版本都从 1 开始、每次部署 +1，`corpLoan` 的版本 3 和另一个 key 的版本 3 互不相干。排查“怎么版本涨这么快”时只看同一个 key 的部署记录，别拿全库的部署次数去套某个 key。

【冷知识】`ACT_HI_DETAIL` 里的变量变更流水只有 `full` 级别才完整：`audit` 下变量只有当前值（`ACT_HI_VARINST`），想查“某个变量是谁、在第几步改成现在这个值的”，必须开 `full`。而且级别调高也不补历史——开 `full` 之前发生的变更永远查不到，审计要追溯旧单时别指望现开级别救场。

【冷知识】历史清理作业不是一条 SQL 全删，而是走 Flowable 的 Batch 机制分批删，进度记在 batch 表里：大库清理期间能看到它一批一批推进，盯 batch 表就知道清到哪了、哪批卡住了。清理跑得慢先查 batch 表的执行情况，别直接怀疑它没跑——分批删本来就是为了不锁表。

## 面试与自测

1. `flowable.history-level` 的四档各记录什么内容？生产上常用哪一档？
2. 同一个 key 每次部署，版本号和旧版本定义会怎样？新实例和在途实例各跑哪一版图？
3. 出问题的新版本想让新单子不再用它，又不想真回滚版本号，正确的做法是什么？

### 参考答案

1. `none` 是什么都不记，无审计能力，生产别用；`activity` 记历史流程实例和活动实例，查得到走过哪些节点、查不到变量；`audit` 再加历史任务、变量、表单等明细，是常用档位，审计够用；`full` 再加变量每次变更的明细（`ACT_HI_DETAIL`），表涨得最快，只有金融审计明确要求时再开。
2. 同一个 key 每次部署版本号自动 +1，旧版本定义不消失、不被覆盖；`startProcessInstanceByKey` 默认用最新版本起新实例，在途实例继续跑它启动时那版图，互不影响。
3. 把旧版本的 BPMN 重新部署一次，它会以最新版本号成为默认启动版本（内容是旧图、版本号是新的），比直接改库里版本号安全得多；要立刻止血也可以先用 `suspendProcessDefinitionById` 挂起出问题的新版本定义，让新单子暂时起不来。注意任何图变更只走部署 API，不要直接改 `ACT_RE_*` 里的定义或版本号救火。

## 关联

- 事件与监听器（汇总节点、死信重试）：[工程实践/Flowable/Flowable-事件与监听器.md](工程实践/Flowable/Flowable-事件与监听器.md)
- 案例（版本升级自然过渡的实战口径）：[项目沉淀/神农-e贷-审批流.md](项目沉淀/神农-e贷-审批流.md)
