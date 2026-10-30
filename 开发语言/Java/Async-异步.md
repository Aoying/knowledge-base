---
知识点: "@Async 异步"
层级: 进阶
适用版本: Spring Boot 2.7
更新日期: 2026-10-31
关联:
  - Java
  - Java-并发编程
  - Spring-Boot
标签:
  - Java
  - 并发
  - 实践
---

# @Async 异步

## 1. 核心概念

`@Async` 是 Spring 的声明式异步：标注的方法经 AOP 代理后被提交到一个 `Executor` 里执行，调用方立即返回。它的价值是把「主流程不必等的活」（发短信、发邮件、记操作日志、推送通知）从请求线程上剥离，缩短接口响应时间。代价是执行脱离了调用方的线程上下文：事务、ThreadLocal、异常感知统统断开，用之前先想清楚这笔账。

## 2. 必须自定义线程池

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Bean("bizExecutor")
    public Executor bizExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(10);
        e.setMaxPoolSize(50);
        e.setQueueCapacity(500);
        e.setThreadNamePrefix("biz-async-");
        e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        e.setWaitForTasksToCompleteOnShutdown(true); // 优雅停机：等任务跑完
        e.setAwaitTerminationSeconds(60);
        e.initialize();
        return e;
    }
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("异步任务异常 {}.{} 参数={}", method.getDeclaringClass().getSimpleName(), method.getName(), Arrays.toString(params), ex);
    }
}
```

```java
@Service
public class NotifyService {
    @Async("bizExecutor")                      // 必须指定线程池名，否则走默认
    public CompletableFuture<Void> sendSms(String mobile) {
        smsClient.send(mobile);
        return CompletableFuture.completedFuture(null);
    }
}
```

- **默认执行器是坑**：不配线程池时 Spring 用 `SimpleAsyncTaskExecutor`，每个任务起一个新线程，无上限——流量一上来线程爆炸 OOM（官方 Javadoc 明确写了不复用线程）。生产必须配 `ThreadPoolTaskExecutor`。
- **线程池参数怎么定**：异步任务多是 IO 型（等下游返回），核心线程数可以给得比 CPU 核数大得多；上限要按下流能承受的并发反推，别让异步把下游打挂。队列满后的拒绝策略见下条。
- 拒绝策略别用默认 `AbortPolicy`（直接抛异常丢任务），IO 型业务一般用 `CallerRunsPolicy` 让调用线程兜底执行，天然限流。

## 3. 经典失效与语义坑

- **同类自调用失效**：跟 `@Transactional` 一个原理，`this.sendSms()` 绕过代理，同步执行。解法：注入自己调，或把异步方法拆到另一个 Bean。
- **void 方法异常会"消失"**：调用方拿不到异常，只能靠 `AsyncUncaughtExceptionHandler` 记日志；要感知结果就返回 `CompletableFuture`。注意返回 `CompletableFuture` 后，异常会包在 future 里，调用方 `get()`/编排时才暴露，编排代码不处理等于照样消失。
- **事务不传播**：`@Async` 方法里拿不到调用方的事务，新线程 = 新事务；需要事务就自己加 `@Transactional`。反过来更隐蔽：主事务还没提交，异步线程就去查刚写的数据，会查不到——「异步读不到主流程刚落库的数据」先怀疑这个时序，而不是怀疑数据库。
- **上下文不传播**：请求线程上的 ThreadLocal（登录用户、链路 traceId、动态数据源 key）默认都不跟随异步线程。需要传递时，在调用前把值取出来作参数传入是最稳的做法；MDC 的 traceId 可用 TaskDecorator 在提交时复制一份过去。动态数据源场景里 `@DS` 要标在异步方法上，见 [多数据源](开发语言/Java/多数据源.md)。
- **`@Async` 标在 private 方法、或类没被 Spring 管理**：代理不生效，静默同步执行，排查时先确认方法确实走了代理。

## 4. CompletableFuture 编排

多个异步任务要并行并汇合时，用 CompletableFuture 而不是裸 @Async：

```java
CompletableFuture<User> f1 = CompletableFuture.supplyAsync(() -> userClient.get(id), bizExecutor);
CompletableFuture<List<Order>> f2 = CompletableFuture.supplyAsync(() -> orderClient.list(id), bizExecutor);
Result r = f1.thenCombine(f2, (u, os) -> new Result(u, os))
            .orTimeout(3, TimeUnit.SECONDS)      // 必须设超时，否则一个慢下游拖死整条链
            .join();
```

- `supplyAsync` 不传执行器会走 ForkJoinPool 公共池，阻塞 IO 别放公共池，永远显式传自己的池。
- 汇合点必须设超时和异常兜底（`exceptionally` 给降级结果），否则异步编排的故障模式是「接口偶发卡死」，比同步还难查。
- 经验分工：**单向通知类**（发完就不管）用 @Async；**并行聚合类**（等多个结果拼返回值）用 CompletableFuture；**任务本身要重试、要落状态、不能丢**的，两者都不够，上消息队列。

## 5. 停机与可观测

- 优雅停机已在配置里开了 `waitForTasksToCompleteOnShutdown`：发布重启时在途异步任务有机会跑完，但超过 `awaitTerminationSeconds` 仍会被中断；发短信这类任务要按「可能执行一半被杀」设计幂等。
- 线程名前缀必须有意义（`biz-async-`），日志里一眼能分清是哪个池的线程；再配合对队列长度、活跃线程数的监控告警，池打满在用户感知前就能发现。


## 小技巧与冷知识

【小技巧】线程池 Bean 叫 taskExecutor 会被 Spring 自动当成默认异步执行器：@Async 不指定池名时，Spring 会先找一个名字叫 taskExecutor 的 TaskExecutor Bean，找到了就用它，找不到才回落到 SimpleAsyncTaskExecutor。项目里只有一个异步池时，把 Bean 名定成 taskExecutor，@Async 上就不用每个方法都写池名；有多个池再显式指定，默认池只留给最通用的那个。

【小技巧】@Async 方法返回 CompletableFuture 时，方法里抛的异常不会进 AsyncUncaughtExceptionHandler：异常会被包进 future 里，handler 只管返回 void 的方法。调用方必须在编排里处理（exceptionally、handle 或 get 时捕获），否则异常悄悄躺在 future 里没人看；想让 handler 生效就别返回 future，两者是互斥的异常通道。

【小技巧】thenApply 和 thenApplyAsync 跑的线程不一样：不带 Async 的回调直接跑在让上游 future 完成的那个线程上，可能是调用方线程、也可能是上游任务的线程；带 Async 的回调才会被提交到指定执行器。想把回调也隔离到业务池，必须写 thenApplyAsync(fn, bizExecutor)，只写 thenApply 等于把回调偷偷塞回了别人的线程。

【小技巧】join 和 get 的异常类型不一样：join 抛 unchecked 的 CompletionException，get 抛 checked 的 ExecutionException 和 InterruptedException。编排链里用 join 写起来顺手，但拿根因要一层层剥 CompletionException.getCause()；日志里只打外层异常，真正的下游报错会被藏一层，排障先剥到 cause 再看。

【小技巧】orTimeout 和 completeOnTimeout 是两种超时策略：orTimeout 超时后抛 TimeoutException，整条链直接失败；completeOnTimeout 超时后用你给的默认值继续往下走。想“超时就降级”用后者，想“超时就报错”用前者；两者都是 Java 9 才加进 CompletableFuture 的，Java 8 的项目照抄会编译不过。

【小技巧】用 TaskDecorator 复制 MDC 时，任务跑完要清掉：装饰器在提交时把调用方的 traceId 拷进异步线程，任务结束时必须在 finally 里 MDC.clear()，否则线程回池后，旧 traceId 会被下一个复用该线程的任务带上，日志里出现“一个 traceId 串了两件不相干的事”。复制和清理是一对，装饰器里只写复制不写清理等于埋了串号的坑。

【冷知识】@Async 是 Spring 3.0 才引入的：2009 年发布的 Spring Framework 3.0 首次带来 @Async 和 @Scheduled 这一对声明式注解，之前想异步只能自己 new Thread 或往 Executor 里 submit。理解这点就明白为什么它和 @Transactional 共享同一套 AOP 代理机制——两者都是 3.0 时代“用注解声明横切关注点”这个思路的产物。

【冷知识】ForkJoinPool 公共池的大小默认是 CPU 核数减 1：CompletableFuture 不传执行器走的就是这个池，机器 8 核它只有 7 个线程。阻塞 IO 的任务一旦占满这 7 个线程，并行流、别的 supplyAsync 全都会被拖住；原文说“阻塞 IO 别放公共池”，根子就是这个池子小到经不起任何一个阻塞任务。

【冷知识】优雅停机只等“已经提交的”任务：setWaitForTasksToCompleteOnShutdown(true) 让容器关闭时等在途任务跑完，但关机流程一旦开始，往池里新提交的任务会被直接拒绝。发布窗口里“最后一批通知还没发完”时，先停掉新任务的入口、再让容器关机，顺序反了就会丢任务；停机保护的是已提交的任务，不是还没提交的。

## 6. 面试与自测

1. 同一个类里 `this.sendSms()` 调用 `@Async` 方法，为什么还是同步执行？怎么修？
2. `@Async` 方法返回 void 时异常为什么会“消失”？要让调用方感知结果该怎么改？
3. 主流程刚落库的数据，异步线程去查却查不到，最可能是什么原因？ThreadLocal 上下文（如 traceId、动态数据源 key）又该怎么传？

### 参考答案

1. `@Async` 靠 AOP 代理生效，`this.` 自调用绕过代理，等于直接调原方法，所以同步执行。修法：注入自己（自注入）再调，或把异步方法拆到另一个 Bean 里调。
2. void 方法没有返回值可承载异常，调用方拿不到，只能靠 `AsyncUncaughtExceptionHandler` 记日志。改返回 `CompletableFuture`，异常会包在 future 里，调用方 `get()`/编排时处理；但编排代码不处理，异常照样等于消失。
3. 异步是新线程、新事务，主事务还没提交时异步线程自然查不到刚写的数据——先怀疑时序，不是数据库。上下文默认不跟随：最稳是调用前把值取出来作参数传入；traceId 可用 TaskDecorator 在提交时复制 MDC；动态数据源要在异步方法上标 `@DS`。

## 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
- [虚拟线程](开发语言/Java/虚拟线程.md)
