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

## 基础知识补强

### 核心概念与术语

- **进程 vs 线程**：进程是资源分配单位，有独立内存；线程是 CPU 调度单位，同一进程内的线程共享堆内存。
- **并发 vs 并行**：并发是多个任务交替推进（单核也能）；并行是多个任务同一时刻真的一起跑（需要多核）。
- **线程安全**：同一份代码被多个线程同时调用，结果仍正确，就叫线程安全；判断标准只看结果对不对。
- **竞态条件（race condition）**：结果取决于线程执行顺序的先后，顺序一变结果就错，是并发 bug 的总根源。
- **临界区**：访问共享资源的代码片段，同一时刻只允许一个线程进入，否则数据会乱。
- **原子性**：一个操作不可分割，要么全做完要么没做；`i++` 不是原子操作（读、加、写三步）。
- **可见性**：一个线程改了共享变量，其他线程能不能马上看到；各线程有自己的缓存，改动可能滞留在缓存里。
- **有序性**：代码实际执行顺序可能与书写顺序不同（编译器与 CPU 会重排序），单线程看不出问题，多线程会踩坑。
- **共享变量**：被多个线程访问的变量（通常是堆上的字段）；只被一个线程用的局部变量天然安全。
- **守护线程（daemon）**：为其他线程服务的后台线程；所有非守护线程结束后，JVM 不等守护线程，直接退出。
- **上下文切换**：线程被换下 CPU 时要保存和恢复现场，线程不是越多越快，切换开销会吃掉收益。
- **happens-before**：判断可见性的规则集；简单记：解锁 happens-before 后续加锁、volatile 写 happens-before 后续读、线程 start/join 有传递性。

### 基础语法与用法

```java
// 创建线程的三种基本写法（理解原理用，业务代码请用线程池）
// 写法 1：继承 Thread，重写 run
class MyThread extends Thread {
    public void run() { System.out.println("跑在: " + Thread.currentThread().getName()); }
}
new MyThread().start();        // 必须调 start()，直接调 run() 只是普通方法调用

// 写法 2：实现 Runnable，把任务和线程解耦（最常用）
Thread t = new Thread(() -> System.out.println("任务逻辑"));
t.start();

// 写法 3：Callable + FutureTask，需要返回值时用
Callable<Integer> job = () -> 42;
FutureTask<Integer> ft = new FutureTask<>(job);
new Thread(ft).start();
System.out.println(ft.get());  // get() 会阻塞等结果
```

```java
// sleep 与 join：等别人先跑完
Thread worker = new Thread(() -> {
    try { Thread.sleep(200); } catch (InterruptedException e) { /* 见下文 */ }
    System.out.println("worker 干完");
});
worker.start();
worker.join();                 // 当前线程等 worker 结束再往下走
System.out.println("主线程最后打印");
```

```java
// 中断：Java 没有"强杀线程"，靠协作式中断
Thread task = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {  // 查中断标志
        // 做一小块工作，然后回头看一眼标志
    }
    System.out.println("收到中断，收尾退出");
});
task.start();
task.interrupt();              // 只是把标志置 true，并让 sleep/wait 抛异常
```

```java
// wait / notify：同一把锁上的等待与唤醒，必须在 synchronized 里用
final Object lock = new Object();
Thread waiter = new Thread(() -> {
    synchronized (lock) {
        try { lock.wait(); } catch (InterruptedException e) { return; }
        System.out.println("被唤醒了");
    }
});
waiter.start();
Thread.sleep(100);
synchronized (lock) { lock.notify(); }   // 唤醒一个等待者；notifyAll 唤醒全部
```

```java
// synchronized 的三种加锁粒度（补基础形态）
class Counter {
    private int n = 0;
    public synchronized void add() { n++; }          // 锁当前对象 this
    public void addBlock() {
        synchronized (this) { n++; }                 // 语句块：只锁关键几行
    }
    public static synchronized void addStatic() { }   // 锁当前类的 Class 对象
    public int get() { return n; }
}
```

### 关键机制速览

- **线程生命周期 6 状态**：NEW（新建）→ RUNNABLE（可运行/运行中）→ BLOCKED（等锁）、WAITING（无限等待）、TIMED_WAITING（限时等待）→ TERMINATED（结束）。
- **状态怎么变**：`sleep`、`wait(timeout)` 进 TIMED_WAITING；`wait()`、 `join()` 进 WAITING；抢锁失败进 BLOCKED。
- **run 与 start**：`start()` 才真正向系统申请新线程，再由 JVM 在新线程里回调 `run()`；重复调用 `start()` 抛异常。
- **sleep 不释放锁**：睡着时手里的锁不松；`wait` 恰好相反，会释放锁并交出等待权。
- **yield 只是建议**：`Thread.yield()` 提示调度器"我可以让一下"，对方理不理看心情，不能当同步手段。
- **中断的真实语义**：中断是对阻塞方法的"敲门"，`sleep/wait/join` 被中断抛 `InterruptedException` 时会清掉中断标志，catch 里通常要恢复标志或直接退出。
- **内存模型一句话**：主内存存共享变量，线程在自己的工作内存里操作副本，靠同步手段（锁、volatile）把修改刷回去并让别人看见。
- **为什么局部变量安全**：局部变量活在各自线程的栈帧里，从不共享，自然不存在竞态。
- **线程组与命名**：给线程起有业务含义的名字（`setName`），排查问题时日志和堆栈一眼能认。
- **优先级作用有限**：`setPriority` 只是给调度器的建议，操作系统可以无视，不要用它保证业务顺序。

### 新手易错点

- **直接调 run()**：`t.run()` 在当前线程里跑完，没有新线程，并发逻辑全失效。
- **共享可变状态不加保护**：多个线程写同一个字段，又没锁没原子类，就是在等偶发 bug。
- **catch 中断异常后吞掉**：捕获 `InterruptedException` 后既不恢复中断标志也不退出，线程收到中断也停不下来。
- **用 sleep 做同步**："睡 1 秒应该好了吧"是最不靠谱的时序假设，快慢取决于机器负载。
- **锁对象随手换**：`synchronized (new Object())` 每次都是新锁，等于没锁；锁对象必须固定且大家共用同一把。
- **在锁里调外部方法**：持锁期间调网络、调回调，极易拉长临界区甚至死锁；临界区只放最必要的几行。
- **静态与实例锁混淆**：`synchronized` 实例方法与静态方法锁的不是同一个对象，互相之间不互斥。
- **以为 volatile 能保计数**：volatile 只保可见和有序，`count++` 的复合动作仍需要锁或原子类。
- **线程结束不等于任务无异常**：`run()` 里抛出的异常默认只打到控制台，不处理就静默丢失重要错误。
- **忽略可见性**：用普通 boolean 做跨线程停止开关，子线程可能永远看不到新值，开关必须 volatile 或用中断机制。
