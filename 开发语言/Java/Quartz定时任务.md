---
知识点: Quartz 定时任务
层级: 进阶
适用版本: Quartz 2.3 / Spring Boot 2.7.3
更新日期: 2026-10-31
关联:
  - Spring-Boot
  - 线程池
  - Flowable
标签:
  - Java
  - 实践
  - 定时任务
---

# Quartz 定时任务

## 1. 核心概念

Quartz 是 Java 生态最老牌的任务调度框架，解决 `@Scheduled` 解决不了的三件事：**任务持久化**（重启不丢）、**集群防重复**（多节点只跑一份）、**运行时动态增删改任务**。神农 e 贷法人版的对账、报表、催办提醒这类业务定时任务就属于这一档。

三个核心角色，关系必须分清：

| 角色 | 是什么 | 关键点 |
| --- | --- | --- |
| `Job` | 任务逻辑，只写一个 `execute` 方法 | 每次触发都新建实例，按无状态写 |
| `JobDetail` | 任务定义：绑哪个 Job、身份（name/group）、参数 | 可被多个 Trigger 引用 |
| `Trigger` | 触发规则：何时跑、重复策略、错过怎么办 | 只绑一个 JobDetail |
| `Scheduler` | 调度器：注册 JobDetail + Trigger 统一驱动 | 应用内一般单例 |

- 参数传递走 `JobDataMap`：JobDetail 和 Trigger 各有一份，取值时 Trigger 的覆盖 JobDetail 的；`JobExecutionContext.getMergedJobDataMap()` 拿合并结果。
- 身份 = name + group，二者组合唯一，动态管理任务全靠它寻址。

## 2. 基础用法

```java
public class ReconcileJob implements Job {
    @Override
    public void execute(JobExecutionContext ctx) {
        String bankCode = ctx.getMergedJobDataMap().getString("bankCode");
        // 对账逻辑……
    }
}

JobDetail job = JobBuilder.newJob(ReconcileJob.class)
        .withIdentity("reconcile", "bank")
        .usingJobData("bankCode", "ABC")
        .storeDurably()
        .build();
Trigger trigger = TriggerBuilder.newTrigger()
        .withIdentity("reconcileTrigger", "bank")
        .startNow()
        .withSchedule(CronScheduleBuilder.cronSchedule("0 30 2 * * ?"))
        .build();
scheduler.scheduleJob(job, trigger);
```

### Cron 表达式：和 Linux cron 不是一回事

- Quartz 是 **6 或 7 段**：`秒 分 时 日 月 周 [年]`，秒打头；Linux 的五段写法直接贴过来必报错。
- `?` 只能用在「日」和「周」两位，表示「不指定」——这两位互斥，一位给了具体值另一位就写 `?`，这是新手第一大错点。
- 特殊符号：`L` 月末/周内最后一天，`W` 最近工作日，`#` 第几个星期几（如 `6#3` 每月第三个周五）。

| 表达式 | 含义 |
| --- | --- |
| `0 0 9 * * ?` | 每天 9:00 |
| `0 0/15 * * * ?` | 每 15 分钟 |
| `0 0 2 ? * MON-FRI` | 工作日凌晨 2:00 |
| `0 0 23 L * ?` | 每月最后一天 23:00 |

## 3. Spring Boot 集成与注入坑

- 引入 `spring-boot-starter-quartz`，Boot 自动装配 `Scheduler`，JobDetail/Trigger 注册为 Bean 即可生效；`spring.quartz.job-store-type` 选 `memory`（默认）或 `jdbc`，jdbc 模式 Boot 可自动建表（`spring.quartz.jdbc.initialize-schema`，生产建议手动执行官方 DDL 并管控版本）。
- **经典空指针坑**：Quartz 默认用反射自己 `new` Job 实例，不经过 Spring 容器，Job 里的 `@Autowired` 全是 null。解法是自定义 JobFactory 让创建走 Spring：

```java
@Component
public class SpringJobFactory extends AdaptableJobFactory {
    @Autowired private AutowireCapableBeanFactory beanFactory;
    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        Object job = super.createJobInstance(bundle);
        beanFactory.autowireBean(job);
        return job;
    }
}
// 配置类里 schedulerFactoryBean.setJobFactory(springJobFactory)
```

- Boot 2.7 用 `QuartzProperties` + 自定义 `SchedulerFactoryBeanCustomizer` 挂 JobFactory 是更顺手的写法，本质同上。

## 4. 集群与持久化

- JDBC JobStore 把触发器状态存进 `QRTZ_*` 系列表（11 张），多节点连同一个库、`org.quartz.jobStore.isClustered=true` 即成集群。
- 防重复原理：节点触发前要抢 `QRTZ_LOCKS` 里的行锁，抢到的才执行——所以集群是**抢占式单跑**，不是负载均衡分片：同一 Trigger 同一时刻只有一个节点执行，想并行分片得自己按数据范围拆多个 JobDetail。
- 集群节点时钟必须同步，时钟漂移会造成重复触发或错过触发，NTP 是前置条件（与链路追踪对时钟的要求同理）。
- 持久化用 OceanBase（MySQL 模式）存 `QRTZ_*` 表没有障碍，按普通 InnoDB 兼容表对待即可，表前缀可通过 `spring.quartz.properties.org.quartz.jobStore.tablePrefix` 修改。

## 5. Misfire：错过触发怎么办

任务该跑时没跑上（应用停机、线程池被长任务占满、暂停太久）即 misfire，超过 `org.quartz.jobStore.misfireThreshold`（默认 60 秒）才算数。CronTrigger 的策略：

| 策略 | 行为 | 用法 |
| --- | --- | --- |
| `FIRE_ONCE_NOW` | 立刻补跑一次，之后按原计划 | 报表、对账：漏了要补，但只补一次 |
| `DO_NOTHING` | 不补，等下个周期 | 催办提醒、状态同步：补跑反而重复打扰 |
| `IGNORE_MISFIRE_POLICY` | 错过几次补几次 | 慎用，停机一周能瞬间补跑上百次打爆下游 |

- Boot 里配置：`CronScheduleBuilder.cronSchedule(expr).withMisfireHandlingInstructionDoNothing()`，别用默认 smart policy 蒙——smart policy 对 Cron 触发器等价于 FIRE_ONCE_NOW，不一定是你想要的。

## 6. 并发控制注解

- `@DisallowConcurrentExecution`（标在 Job 类上）：上一次没跑完，下一次触发排队等待而不是并发再起一个。对账、批处理必备，否则慢任务会自己压死自己。
- `@PersistJobDataAfterExecution`：把 execute 里对 JobDataMap 的修改写回（常与上一个注解搭配做计数器/游标推进）；不加则每次都是初始值。
- 长任务还会占满 Quartz 自己的工作线程池（`org.quartz.threadPool.threadCount`，Quartz 默认为 10）：线程被长任务占光后其他任务全部迟到并触发 misfire。长耗时任务要么调大线程数，要么任务里只做「投递」动作，把重活丢给业务线程池异步执行，见 [线程池](开发语言/Java/并发/线程池.md)。

## 7. 与 @Scheduled 的取舍

| 维度 | Spring `@Scheduled` | Quartz |
| --- | --- | --- |
| 上手成本 | 一个注解 | 依赖 + 配置 + 建表（jdbc 模式） |
| 持久化 | 无 | JDBC 存储，状态可查可恢复 |
| 集群 | 每节点各跑一份（需另配 ShedLock 防重） | 原生集群单跑 |
| 动态修改 | 改代码重启 | API/DB 增删改、暂停恢复 |
| 失败补偿 | 无 | misfire 策略 |

- 选型口径：单机、无状态、漏跑无所谓的任务用 `@Scheduled`；涉及资金、要审计、要集群、要运营侧可管理，一律 Quartz。中间地带（集群防重但不需要持久化）可以 `@Scheduled` + ShedLock，别为了防重硬上全套 Quartz。
- 与 Flowable 分工：Flowable 流程里的定时事件（边界定时器、延时节点）归 Flowable 自己的 Job 执行器管，**不要**再用 Quartz 去驱动流程；Quartz 只管流程之外的业务定时任务。两者共存时各自独立，别混表、别互相调用调度 API。见 [Flowable](工程实践/Flowable.md)。

## 8. 疑难杂症

- **任务在集群里重复执行**：先查 `isClustered` 是否真开启（共用库但没开集群 = 每节点独立调度，必重复）；再查是否有两套应用连了同一个库各自调度。
- **表达式对但就是不跑**：按顺序查——Trigger 被 pause、身份 group 写错、cron「日/周」两位同时给了具体值、时区不是 Asia/Shanghai（容器默认 UTC 让「凌晨 2 点」变北京时间 10 点）。时区在 Trigger 上显式 `inTimeZone(TimeZone.getTimeZone("Asia/Shanghai"))` 或统一 JVM 时区，别靠部署环境碰运气。
- **重启后任务消失**：memory JobStore 的必然结果，要么换 jdbc，要么靠启动时代码注册并保证幂等。

## 小技巧与冷知识

【小技巧】cron 表达式上线前先用 `CronExpression.isValidExpression(expr)` 静态方法校验一次：它在调度前就能拦住非法表达式，比等 `scheduler.scheduleJob` 抛异常再排查要早。动态录入 cron（运营后台填表达式）的场景必加这道校验，否则一条写错的表达式要到触发时才暴露，中间还可能静默不跑。

【小技巧】`storeDurably()` 不是可有可无的修饰：没加它的 JobDetail 在最后一个 Trigger 被删掉时会被调度器自动清理掉，加了才会留下来等新 Trigger 挂。动态增删 Trigger 的任务一定要配 durable，否则删一次触发器、任务定义就跟着没了，下次想重建还得重新注册 JobDetail。

【小技巧】JDBC 模式下 JobDataMap 里只放 String、数字这类可序列化值：持久化时 JobDataMap 是序列化后存进数据库的，放进一个没实现 `Serializable` 的业务对象，建任务时直接报错。参数能用字符串表达就用字符串，复杂对象只传 ID、执行时再查库，别把对象本身塞进 JobDataMap。

【小技巧】Job 里抛 `JobExecutionException` 可以控制“这次失败后怎么办”：`setRefireImmediately(true)` 让调度器立刻重跑一次，适合偶发失败（如下游短暂抖动）；`setUnscheduleAllTriggers(true)` 则直接把该任务的所有触发器都摘掉，适合“再跑也没用、需要人工介入”的坏数据。直接抛普通 RuntimeException 是没有这些语义的，失败就只记一条日志。

【小技巧】想中途停掉一个正在跑的长任务，Job 要实现 `InterruptableJob` 接口：它多一个 `interrupt()` 方法，调度器调 `scheduler.interrupt(jobKey)` 时会回调它，由你在里面置标志位、让 execute 里的循环尽早退出。普通 Job 没有这个入口，调度器只能等它自己跑完，`pauseJob` 也只能拦住下一次触发、停不掉正在跑的这一次。

【小技巧】应用停机时调 `scheduler.shutdown(true)` 而不是 `shutdown(false)`：传 true 会等正在执行的任务跑完再关，传 false 立刻返回、不管手头的任务。滚动发布、优雅停机走 true 这条路，正在跑的对账任务不会被半路掐断；但要先确认任务有超时兜底，否则一个死循环任务会把停机无限拖住。

【小技巧】“每天 9 点到 17 点、每 15 分钟跑一次”这种带时间窗口的规则，别硬拼 cron：用 `DailyTimeIntervalTrigger`，它天生就是“每天某个时间段内按间隔触发”，再配 `HolidayCalendar` 还能把法定节假日排除掉。cron 写这种窗口要写一串表达式还容易漏边界，专用触发器加日历才是正路。

【小技巧】暂停要分清 `pauseTrigger` 和 `pauseJob` 的范围：前者只停某一个触发器，后者停掉该 Job 下的全部触发器；恢复时 `resumeJob` 会把该 Job 下所有触发器一起恢复，包括你之前单独 pause 的那个。只想临时停某一条触发规则时用 pauseTrigger，想整组停再用 pauseJob，恢复前先确认有没有“本想继续停着”的触发器被顺带恢复了。

【小技巧】关键任务在 JobDetail 上开 `requestRecovery(true)`：节点崩溃时正在跑的任务，调度器恢复后会重新触发一次，相当于给“跑一半就宕机”留了后手。默认是不开的，崩溃时跑一半的任务直接按 misfire 逻辑走，不一定补跑；对账、资金类任务值得显式打开，非关键任务则没必要。

【冷知识】集群节点之间靠 `clusterCheckinInterval` 互相“报到”，默认 7500 毫秒：每个节点每隔这段时间往 `QRTZ_SCHEDULER_STATE` 表写一次心跳，超过间隔没报到的节点会被其他节点判定为挂了，它手里正在跑的任务会被接管重跑。把这个间隔设得太大，故障接管就慢；设得太小，心跳写库本身就成了负担。

【冷知识】`withIdentity("reconcile")` 只写名字不写 group 时，group 默认是 `"DEFAULT"`：`Scheduler.DEFAULT_GROUP` 就是这个字符串，所有没显式分组的 Job 和 Trigger 都落在同一个默认组里。按 name + group 寻址动态管理任务时，不知道这个默认值，查出来的任务可能比预期的多——不同业务的任务最好显式分组，别都挤在 DEFAULT 里。

【冷知识】`SimpleTrigger` 的无限重复用常量 `REPEAT_INDEFINITELY`，它的值就是 -1：`withRepeatCount(SimpleTrigger.REPEAT_INDEFINITELY)` 和直接写 -1 等价，但读代码时常量一眼能看出意图。Quartz 里不少“无限”都是用 -1 表达的，看到 -1 先别当成写错了，对照一下是不是这个常量。

## 9. 面试与自测

- JobDetail 与 Trigger 为什么要分开？一对多关系怎么理解？
- Quartz 集群如何保证同一任务只跑一个节点？是分片吗？
- misfire 是什么？三种策略分别适合什么任务？
- Job 里注入 Spring Bean 为 null 的原因与解法？
- `@DisallowConcurrentExecution` 解决什么问题？和分布式锁防重有何区别（前者只在同一 JobDetail 维度、依赖调度器，后者跨系统）？

### 参考答案

1. 因为二者职责不同、是一对多关系：JobDetail 是任务定义，绑哪个 Job、身份（name + group）、参数；Trigger 是触发规则，何时跑、重复策略、错过怎么办，只绑一个 JobDetail。分开后同一个 JobDetail 可被多个 Trigger 以不同时间规则引用，也便于按 name + group 寻址做动态增删改。
2. 靠 JDBC JobStore + 集群抢锁保证：多节点连同一个库、开启 org.quartz.jobStore.isClustered=true，节点触发前要抢 QRTZ_LOCKS 里的行锁，抢到的才执行，同一 Trigger 同一时刻只有一个节点执行。这不是分片负载均衡，想并行要自己按数据范围拆多个 JobDetail；集群节点时钟还必须用 NTP 同步，否则会重复或错过触发。
3. misfire 是任务该跑时没跑上（停机、线程池被长任务占满、暂停太久），超过 misfireThreshold（默认 60 秒）才算数。FIRE_ONCE_NOW 立刻补跑一次、之后按原计划，适合报表、对账这类漏了要补、但只补一次的任务；DO_NOTHING 不补、等下个周期，适合催办提醒、状态同步这类补跑反而重复打扰的任务；IGNORE_MISFIRE_POLICY 错过几次补几次，慎用，停机一周可能瞬间补跑上百次打爆下游。注意 Cron 触发器的默认 smart policy 等价于 FIRE_ONCE_NOW，不要蒙着用。
4. 因为 Quartz 默认用反射自己 new Job 实例，不经过 Spring 容器，所以 Job 里的 @Autowired 全是 null。解法是自定义 JobFactory 让创建走 Spring，如继承 AdaptableJobFactory、重写 createJobInstance 后用 AutowireCapableBeanFactory.autowireBean 注入，再挂到 SchedulerFactoryBean 上；Boot 2.7 也可用 QuartzProperties + SchedulerFactoryBeanCustomizer 实现，本质相同。
5. 它解决同一 JobDetail 的并发自压问题：标在 Job 类上后，上一次没跑完、下一次触发排队等待而不是并发再起一个，对账、批处理必备，否则慢任务会自己压死自己。区别在于作用域：它只在同一 JobDetail 维度、依赖调度器生效；分布式锁防重是跨系统的，不依赖同一调度器、能覆盖多系统重复执行的场景。

## 10. 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)：自动装配与 starter 机制，quartz starter 的配置入口。
- [线程池](开发语言/Java/并发/线程池.md)：Quartz 工作线程池与业务线程池的隔离。
- [Flowable](工程实践/Flowable.md)：流程内定时事件的自有调度，与 Quartz 的分工边界。
