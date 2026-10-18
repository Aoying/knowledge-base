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
