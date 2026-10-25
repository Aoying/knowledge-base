---
知识点: @Async 异步
层级: 进阶
适用版本: Spring Boot 2.7
更新日期: 2026-10-23
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

## 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
- [虚拟线程](开发语言/Java/虚拟线程.md)
