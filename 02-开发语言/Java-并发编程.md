# Java 并发编程

## synchronized 与 volatile

- `synchronized`：互斥 + 可见性；方法锁 this，静态方法锁 Class。
- `volatile`：只保证可见性 + 禁止重排序，不保证原子性（`count++` 照样丢）。
- 双重检查单例：`volatile` 必加，不然可能拿到半初始化对象。

## JUC 工具

| 工具 | 一句话 |
|---|---|
| ReentrantLock | 可重入锁，比 synchronized 多 tryLock / 公平锁 |
| CountDownLatch | 等 N 件事做完（如 3 个接口都返回） |
| CyclicBarrier | N 个线程在栅栏处会合再一起走 |
| Semaphore | 限流，允许 N 个并发 |
| ReadWriteLock | 读多写少 |

## 线程池（背参数）

```java
new ThreadPoolExecutor(
    5, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadPoolExecutor.CallerRunsPolicy());
```

- 7 参数：核心 / 最大 / 空闲存活 / 单位 / 队列 / 工厂 / 拒绝策略。
- 拒绝策略：Abort（抛异常）/ CallerRuns（调用者自己跑，天然限流）/ Discard。
- IO 密集：`2*CPU+1`；计算密集：`CPU+1`。别再无脑 `Executors.newFixedThreadPool`（队列无界会 OOM）。

## 并发容器

- `ConcurrentHashMap`：分段锁 / CAS，`computeIfAbsent` 原子初始化。
- `CopyOnWriteArrayList`：读多写极少；`BlockingQueue`：生产者消费者。

## ThreadLocal

- 每个线程一份拷贝：用户信息、SimpleDateFormat（线程不安全，用它包）。
- 用完 `remove()`！线程池复用线程，不清就是内存泄漏。

## 和虚拟线程的衔接

- 虚拟线程（Java 21+）让"一个请求一个线程"成为现实，见 [02-开发语言/Java.md](02-开发语言/Java.md)。
- 注意：`synchronized` 块里调阻塞方法会 pin 住载体线程，虚拟线程下优先用 ReentrantLock。

## CompletableFuture：异步编排

```java
CompletableFuture.supplyAsync(() -> userService.get(id), pool)
    .thenApplyAsync(u -> orderService.list(u.getId()), pool)
    .exceptionally(e -> { log.error("查询失败", e); return List.of(); })
    .orTimeout(3, TimeUnit.SECONDS)
    .join();
```

- 永远传自定义线程池：默认 `ForkJoinPool.commonPool` 是守护线程 + 不可控，业务一挂它就悄悄停。
- `allOf` 等多个并行任务（现代版 CountDownLatch）；`anyOf` 取最快的一个。
- 异常链：`exceptionally` 兜底 / `handle` 拿结果+异常 / `whenComplete` 只记录不改结果。

## 原子类与 LongAdder

- `AtomicInteger` 靠 CAS 自旋：竞争不激烈时比锁快；激烈时自旋烧 CPU，不如锁。
- 高并发计数用 `LongAdder`（分段累加，最后求和）：统计、监控、QPS 计数首选。

## 死锁排查

- `jstack <pid> | grep -A5 BLOCKED`，或 Arthas `thread -b` 一键定位死锁线程。
- 预防三招：锁按固定顺序获取；能用 `tryLock(timeout)` 就别裸 `lock()`；锁粒度能小就小。

## AQS 一句话

- `ReentrantLock` / `CountDownLatch` / `Semaphore` / `ReadWriteLock` 底层都是 AQS：一个 `state` 整数 + CLH 等待队列。
- 懂 AQS（独占 vs 共享、公平 vs 非公平），JUC 工具就通了，不用一个个背。
