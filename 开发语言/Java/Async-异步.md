---
知识点: @Async 异步
层级: 进阶
适用版本: Spring Boot 2.7
更新日期: 2026-10-22
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

## @Async 异步实战与坑

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
- **同类自调用失效**：跟 `@Transactional` 一个原理，`this.sendSms()` 绕过代理，同步执行。解法：注入自己调，或把异步方法拆到另一个 Bean。
- **void 方法异常会"消失"**：调用方拿不到异常，只能靠 `AsyncUncaughtExceptionHandler` 记日志；要感知结果就返回 `CompletableFuture`。
- **事务不传播**：`@Async` 方法里拿不到调用方的事务，新线程 = 新事务；需要事务就自己加 `@Transactional`。
- 拒绝策略别用默认 `AbortPolicy`（直接抛异常丢任务），IO 型业务一般用 `CallerRunsPolicy` 让调用线程兜底执行，天然限流。

