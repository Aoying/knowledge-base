---
知识点: Spring 事务
层级: 进阶
适用版本: Spring Boot 2.7.3 / Spring Framework 5.3
更新日期: 2026-10-31
关联:
  - Spring-Boot
  - Spring-IOC与AOP
  - Flowable
  - 多数据源
标签:
  - Java
  - 数据库
  - 原理
  - 实践
---

# Spring 事务

## 1. 核心概念

Spring 事务是对 JDBC / JPA / MyBatis 等资源事务的统一抽象：业务代码只声明「哪里是事务边界、什么情况回滚」，开启、提交、回滚由框架代理完成。解决的核心问题是多步写操作（扣额度 + 写流水 + 推流程）的原子性，避免写一半失败留下脏数据。声明方式以 `@Transactional` 注解为主，少数需要动态边界的场景用编程式 `TransactionTemplate`。

## 2. 基础知识

```java
@Service
public class LoanApplyService {

    @Transactional(rollbackFor = Exception.class)
    public void submit(ApplyDTO dto) {
        applyMapper.insert(dto.toEntity());      // 业务表
        quotaService.deduct(dto.getCustomerId(), dto.getAmount()); // 额度
        // 任一步抛异常，整个方法回滚
    }
}
```

常用属性：

| 属性 | 说明 | 默认 |
|---|---|---|
| `propagation` | 传播行为 | `REQUIRED` |
| `rollbackFor` / `noRollbackFor` | 指定回滚 / 不回滚的异常 | 只回滚 RuntimeException 与 Error |
| `readOnly` | 只读提示，给驱动与 ORM 做优化 | `false` |
| `timeout` | 超时秒数，超时回滚 | -1（不限制） |
| `isolation` | 隔离级别 | 用数据库默认（OceanBase MySQL 模式默认读已提交） |
| `transactionManager` | 多数据源时指定管理器 | 主管理器 |

## 3. 进阶知识

### 传播行为

| 传播行为 | 行为 | 典型用途 |
|---|---|---|
| `REQUIRED` | 有事务加入，没有新建 | 默认，绝大多数业务方法 |
| `REQUIRES_NEW` | 挂起当前事务，永远新建独立事务 | 审计日志、操作流水：主事务回滚也要留痕 |
| `NESTED` | 同一事务内开保存点（savepoint），内层回滚只回滚到保存点 | 批量处理中单条失败不拖垮整批 |
| `SUPPORTS` | 有就加入，没有就非事务执行 | 查询类方法 |
| `NOT_SUPPORTED` | 挂起当前事务，非事务执行 | 不需要事务的大报表导出 |
| `MANDATORY` | 必须已有事务，否则抛异常 | 约束只能被事务方法调用的内部组件 |
| `NEVER` | 必须无事务，有则抛异常 | 极少用 |

两个容易混的点：

- `REQUIRES_NEW` 内层是**独立连接、独立提交**：内层已提交后外层再回滚，内层不会跟着回滚；且内外同时占用两个连接，连接池太小、并发又高时会互相等连接卡死。
- `NESTED` 与外层共用一个连接、一个事务，只是多一个保存点；驱动必须支持 savepoint（OceanBase MySQL 模式支持），且只有最外层提交才真正提交。

### 与 Flowable 流程事务的衔接

Flowable 在 Spring 环境下使用 `SpringTransactionInterceptor`，把流程命令（`startProcessInstance`、`complete` 等）绑到当前 Spring 事务上，前提是 Flowable 的 `ProcessEngineConfiguration` 用的事务管理器与业务一致（`setTransactionManager(...)` 注入同一个 `PlatformTransactionManager`）：

```java
SpringProcessEngineConfiguration config = new SpringProcessEngineConfiguration();
config.setDataSource(dataSource);
config.setTransactionManager(transactionManager); // 与业务同一个
config.setDatabaseSchemaUpdate(ProcessEngineConfiguration.DB_SCHEMA_UPDATE_TRUE);
```

效果：业务方法上的 `@Transactional` 覆盖范围内，流程推进（写 ACT_* 表）与业务表写入**同事务提交、同事务回滚**——审批通过时业务状态更新失败，流程任务也不会错误地往下走。这是神农审批流一致性的根基，别用两个数据源/两个管理器把它们拆开。

注意边界：

- 事务只覆盖**一次命令执行**：流程到达等待状态（用户任务、定时器）时当前命令事务即提交，不是整个流程生命周期一个大事务。
- Flowable 异步执行器在独立线程、独立事务中跑；异步失败不会回滚已提交的业务数据，靠失败重试与业务补偿处理。
- Flowable 表与业务表不在一个库时，单机事务管不了跨库原子性，只能 XA 或最终一致 + 对账补偿，别误以为加了注解就跨库安全。

## 4. 原理剖析

声明式事务 = AOP 代理 + 事务拦截器：

1. Spring 为标注 `@Transactional` 的 Bean 生成代理（机制见 [Spring IOC 与 AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)）。
2. 外部调用先进入 `TransactionInterceptor`，按注解配置通过 `PlatformTransactionManager` 开启事务。
3. `DataSourceTransactionManager` 取连接、关自动提交，并把连接绑定到当前线程的 `ThreadLocal`（`TransactionSynchronizationManager`）；MyBatis 的 `SqlSessionTemplate` 每步都从线程绑定取同一连接，所以能同事务。
4. 正常返回 → 提交；抛异常且命中回滚规则 → 回滚；最后解绑归还连接。

由此推出三条铁律：**事务只存在于当前线程、只经过代理才有事务、连接跟着线程走**。失效场景基本都是这三条的推论。

## 5. 实战应用

- 事务边界放在 Service 层，不放 Controller、不放 Mapper；一个公共方法 = 一个用例 = 一个事务边界。
- 只读查询标 `readOnly = true`：给驱动只读提示，也防止误写。
- 批量导入用 `NESTED` 或分批提交：一条脏数据只回滚本条/批，错误清单单独落表。
- 记流水表用 `REQUIRES_NEW` 独立 Bean 承载（见下方自调用问题），主业务回滚时流水仍在，方便追责排查。
- 事务方法里禁止远程调用、发 MQ 等慢操作：长事务占着连接和行锁，高并发下先爆的是连接池。

## 6. 疑难杂症

### 事务失效场景大全

| 场景 | 原因 | 解法 |
|---|---|---|
| 同类自调用 `this.b()` | 绕过代理 | 注入自身代理 / `AopContext.currentProxy()`（需 `exposeProxy=true`）/ 把 b 拆到另一个 Bean |
| 方法非 `public` | 代理拦截不到（默认） | 改 public，或开启 AspectJ 织入 |
| 方法 `final` / 类 `final` | CGLIB 无法生成子类代理 | 去掉 final |
| 异常被 catch 吞掉 | 拦截器看不到异常，以为成功提交 | 重新抛出，或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| 抛 checked 异常未配 `rollbackFor` | 默认只回滚运行时异常 | `rollbackFor = Exception.class` 作为团队默认写法 |
| 对象不是 Spring Bean（自己 `new` 的） | 没有代理 | 交给容器管理 |
| `@Async` / 新开线程里的方法 | 事务上下文绑在原线程 ThreadLocal | 异步段自己开新事务，且明确它与主事务独立 |
| 多数据源未指定 `transactionManager` | 用了默认管理器，管的是另一个库 | 注解上显式指定，或 `@Primary` 规划主管理器 |
| 表引擎不支持事务（如 MyISAM） | 存储层没有事务能力 | 换 InnoDB / OceanBase 表 |

### 只回滚了一半

现象：方法里两张表，一张回滚一张没回滚。排查顺序：两张表是否同库同连接（多数据源？）、第二张表的写入是否走了另一个 Bean 且对方标了 `REQUIRES_NEW`、是否有一张是流程引擎表走了别的事务管理器。

## 7. 常见误区

- 类上加 `@Transactional` 就以为全类方法都有事务：private 方法、自调用照样失效。
- 把 `rollbackFor` 缺省当保险：业务异常多是 checked 风格自定义异常时，异常抛了数据却提交了，是最隐蔽的事故。
- 认为 `@Transactional` 能管分布式、能约束外部系统：跨库跨服务要分布式事务或最终一致；「外部成功才提交」要走状态机 + 对账/补偿。

## 8. 性能优化

- 缩小事务范围：参数组装、校验、查字典等只读前置动作移出事务方法；`timeout` 给批量任务设上限，避免失控长事务堵住连接池。
- OceanBase 下关注长事务对 undo 的压力，批处理分片提交（每 N 条一事务）。

## 小技巧与冷知识

【小技巧】排障时先确认当前到底有没有事务：`TransactionSynchronizationManager.isActualTransactionActive()` 返回 true 才说明当前线程真的开着事务。方法上明明标了 `@Transactional` 却怀疑没生效时，在方法里打一行这个判断，比翻代理类、翻日志快得多；返回 false 就按原文第 6 节的失效表逐项排查，别先去调传播行为。

【小技巧】`isolation` 只在新建事务时生效，加入已有事务时会被忽略：外层已经是 REQUIRED 的事务，内层方法再标 `isolation = SERIALIZABLE` 不会升级隔离级别，内层注解的隔离级别直接作废。想让某个查询跑在更高隔离级别下，必须让它走一个新建事务的入口（如 REQUIRES_NEW），而不是在同一个事务里改注解。

【小技巧】`timeout` 是整个事务的截止时间，不是单条 SQL 的超时：超时到点时事务会被标记为仅回滚，方法体本身不会被中途打断，走到提交时直接回滚。它不等同于 JDBC 的 `queryTimeout`，后者是单条 SQL 的执行上限；批量任务既要防长事务占连接，timeout 要设，也要给慢 SQL 单独设查询超时，两层都得有。

【小技巧】测试里 `@Transactional` 会让测试方法跑在事务里、结束后自动回滚，数据库不留脏数据；想留数据排查时在测试方法上加 `@Rollback(false)`。这是 Spring TestContext 的特性，不是业务代码的事务——生产代码里别照抄 `@Rollback`，它只在测试里有效。

【小技巧】事务提交后再发消息、再调外部接口，用 `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`：方法里 `publishEvent` 只是把事件记下来，等当前事务提交成功后才真正执行监听器。原文第 5 节说事务里禁止发 MQ，正确姿势是把“发”这一步挂到提交后，事务回滚时事件根本不会发出去，也就不用写补偿去撤消息。

【小技巧】编程式事务别只会用 `TransactionTemplate.execute`：有返回值用 `TransactionCallback<T>`，没返回值用 `TransactionCallbackWithoutResult`，后者省掉一个无意义的 return null。动态决定边界、循环里分批提交的场景用编程式比注解灵活，但模板对象要注入对应的事务管理器，多数据源下别拿错了管理器去构造模板。

【小技巧】`NESTED` 在 JTA 下不支持：保存点是 JDBC 驱动的能力，JTA 全局事务没有保存点概念，内层标 NESTED 会直接抛 `NestedTransactionNotSupportedException`。单库用 JDBC 事务管理器时 NESTED 是分批容错的利器，换成 JTA 或 XA 场景要改回分批提交，别指望保存点跟着跨过去。

【小技巧】事务方法里查自己刚写的数据能读到，但别的连接还读不到：同一事务内是同一连接，自己的未提交写入对自己可见；其他线程、其他连接按读已提交隔离级别看不到这批数据。排障时在事务里打日志查库看到数据，别误以为已经提交入库了——提交前它只对当前事务可见。

【冷知识】`readOnly = true` 只是给框架和驱动的提示，不是数据库层面的强制只读：MySQL/OceanBase 下它主要让驱动和 ORM 少做脏检查、跳过 flush，写操作照样能执行成功。别拿 readOnly 当“防误写”的安全锁，真正的防误写靠权限和代码审查；它的价值是优化提示，不是访问控制。

【冷知识】`@Transactional` 注解是 Spring 2.0（2006 年）才有的，Spring 1.2 时代的声明式事务只能用 XML 的 `<tx:advice>` + 切面配置。注解出现之前，事务边界是配在 XML 里的，改一个方法的事务属性要改配置文件；现在看 XML 方式觉得啰嗦，它却是声明式事务的起点。

【冷知识】`TransactionStatus.isNewTransaction()` 能告诉你当前事务是不是新建的：加入外层事务时返回 false，新建时返回 true。调试传播行为时在方法里打这个值，比猜“到底是加入还是新建”可靠；配合 `isActualTransactionActive()` 一起用，REQUIRED、REQUIRES_NEW、NESTED 的实际行为一眼就能分清。

## 9. 面试与自测

1. 七种传播行为分别什么语义？`REQUIRES_NEW` 与 `NESTED` 的本质区别是什么？
2. `@Transactional` 失效的场景你能列几个？分别为什么？
3. Spring 事务为什么绑线程？异步方法里事务为什么没有？
4. Flowable 与业务表同事务提交是怎么做到的？异步执行器为什么不在这个事务里？
5. 一个事务方法里调了耗时 3 秒的外部接口，会带来什么连锁反应？

### 参考答案

1. REQUIRED：有则加入、无则新建，默认；REQUIRES_NEW：挂起当前事务、永远新建独立事务；NESTED：同一事务内开保存点，内层回滚只到保存点；SUPPORTS：有则加入、无则非事务；NOT_SUPPORTED：挂起当前事务、非事务执行；MANDATORY：必须已有事务否则抛异常；NEVER：必须无事务、有则抛异常。本质区别：REQUIRES_NEW 是独立连接、独立提交，内层提交后外层再回滚也不会连带回滚，且内外同时占两个连接；NESTED 与外层共用一个连接、一个事务，只有最外层提交才真正提交，内层失败只回滚到保存点，且驱动须支持 savepoint。
2. 常见失效场景及原因：同类自调用绕过代理；方法非 public，代理默认拦截不到；类或方法 final，CGLIB 无法生成子类代理；异常被 catch 吞掉，拦截器以为成功而提交；抛 checked 异常未配 rollbackFor，默认只回滚 RuntimeException 与 Error；对象是自己 new 的、不是 Spring Bean，没有代理；@Async 或新线程里的方法，事务上下文不在该线程；多数据源未指定 transactionManager，用错了管理器；表引擎如 MyISAM 本身不支持事务。
3. 因为 DataSourceTransactionManager 把连接关掉自动提交后绑定到当前线程的 ThreadLocal（TransactionSynchronizationManager），MyBatis 的 SqlSessionTemplate 每步都从线程绑定取同一连接，才能同事务。异步方法换了线程，原线程的绑定跟不过去，所以没有事务；异步段要自己开新事务，并明确它与主事务相互独立。
4. Flowable 在 Spring 环境下用 SpringTransactionInterceptor 把流程命令绑到当前 Spring 事务，前提是 ProcessEngineConfiguration 注入与业务同一个 PlatformTransactionManager。这样业务方法 @Transactional 范围内，流程 ACT_* 表写入与业务表写入同事务提交、同事务回滚。但事务只覆盖一次命令执行，到等待状态即提交；异步执行器在独立线程、独立事务中跑，不在该事务里，异步失败不会回滚已提交的业务数据，靠失败重试与业务补偿处理。
5. 会把事务拖长：长事务一直占着数据库连接和行锁，高并发下先爆的是连接池，后续请求排队等待连接，延迟放大、甚至互相等连接卡死；同时长事务还会给 OceanBase 的 undo 带来压力。因此事务方法里禁止远程调用、发 MQ 等慢操作，应把耗时操作移出事务，或缩小事务范围、给批量任务设 timeout 上限。

## 10. 关联知识

- 前置：[Spring Boot](开发语言/Java/Spring/Spring-Boot.md)（注解与数据源配置）
- 相关：[Spring IOC 与 AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)（代理机制）、[多数据源](开发语言/Java/多数据源.md)、[Flowable](工程实践/Flowable.md)（流程引擎事务整合）
- 延伸：分布式事务（Seata）、事务性消息与最终一致
