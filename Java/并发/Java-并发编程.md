---
知识点: Java 并发编程
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-22
关联:
  - Java知识点地图
  - Java
  - 操作系统-进程线程
标签:
  - Java
  - 并发
  - 原理
  - 实践
---

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

- 虚拟线程（Java 21+）让"一个请求一个线程"成为现实，见 [Java/Java.md](Java/Java.md)。
- 注意：`synchronized` 块里调阻塞方法会 pin 住载体线程，虚拟线程下优先用 ReentrantLock。

## CompletableFuture（异步编排）

```java
CompletableFuture<String> a = CompletableFuture.supplyAsync(() -> callA());
CompletableFuture<String> b = CompletableFuture.supplyAsync(() -> callB());
String result = a.thenCombine(b, (x, y) -> x + y).join();  // 两个都完再合并
```

- 代替"开线程 + join"的写法；`thenApply`（转换）、`thenCompose`（再调异步）、`exceptionally`（兜底）。
- 默认用 ForkJoinPool.commonPool，IO 密集传自己的线程池：`supplyAsync(fn, myPool)`。
- 调多个下游接口聚合时用它，别再一个个串行调。

## 原子类与 LongAdder

- `AtomicInteger`：CAS 自旋，竞争不激烈时比锁快。
- `LongAdder`：高并发计数器（分段累加），统计 QPS/计数用它，不用 AtomicLong。
- `AtomicReference` + 不可变对象：无锁更新复杂状态。

## 死锁排查

```bash
jstack <pid> | grep -A 5 "deadlock"   # 直接定位
```

- 线上先 `jstack` 拿现场，再看"哪个线程拿着哪把锁在等哪把锁"。
- 预防：全局固定加锁顺序；能用 `tryLock(超时)` 就别无限等。

## AQS（理解 JUC 的钥匙）

- ReentrantLock / CountDownLatch / Semaphore 底层都是 AQS：一个 `state` 整数 + CLH 等待队列。
- 面试一句话：加锁 = CAS 改 state，抢不到就进队列 park；释放 = 改 state + unpark 下一个。
