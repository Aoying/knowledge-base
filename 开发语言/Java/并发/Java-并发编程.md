---
知识点: Java 并发编程
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
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

> 本篇是并发知识的汇总与索引：核心子主题已拆为独立文件，存放在 `开发语言/Java/并发/` 子目录。先读本篇的全景与精华建立坐标系，再按问题去对应子篇深挖。

## 并发全景导读

并发编程要解决的问题只有一句：多个线程读写同一份状态时，如何保证结果正确，同时别把性能让锁吃光。围绕这句话，知识分四层：

1. **语义层**：什么是线程安全、竞态从哪来——原子性、可见性、有序性三大问题，以及判断可见性的 happens-before 规则。入口：[volatile与JMM](开发语言/Java/并发/volatile与JMM.md)。
2. **工具层**：锁（互斥）、原子类（无锁）、线程池（执行载体）、ThreadLocal（不共享的另一条路）。这是日常编码真正动手的部分。
3. **底座层**：[AQS](开发语言/Java/并发/AQS.md) 与 CAS 撑起整个 JUC——读懂这一层，所有同步工具的源码都变成同一套模板的填空题。
4. **容器层**：[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md) 等并发容器，把上面的机制封装成现成的数据结构。

学习顺序建议：本篇精华 → volatile与JMM → synchronized与锁 → 线程池 → CAS与原子类 → AQS → ThreadLocal；容器篇可随时插入。

## 独立文件索引

- [线程池](开发语言/Java/并发/线程池.md)：七参数与执行流程、拒绝策略、为什么禁用 Executors、CompletableFuture 的池要自己传。
- [synchronized与锁](开发语言/Java/并发/synchronized与锁.md)：内置锁与 ReentrantLock 的分工、读写锁、死锁四条件与 jstack 排查。
- [volatile与JMM](开发语言/Java/并发/volatile与JMM.md)：happens-before 五条核心规则、DCL 为什么必须 volatile、与 JVM 内存区域的区别。
- [CAS与原子类](开发语言/Java/并发/CAS与原子类.md)：乐观锁的自旋骨架、ABA 与版本号、LongAdder 为什么比 AtomicLong 快。
- [AQS](开发语言/Java/并发/AQS.md)：一个 state + 一条等待队列如何撑起 Lock、信号量与倒计时闩。
- [ThreadLocal](开发语言/Java/并发/ThreadLocal.md)：每线程一份副本的原理、与线程池共用时的串数据与泄漏、TTL 跨池传递。

相关但不在本目录的两篇：

- [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)：并发容器代表作，分段与 CAS 的组合落地，归在集合子目录。
- [虚拟线程](开发语言/Java/虚拟线程.md)：Java 21 的轻量线程，「一个请求一个线程」回归；注意其下 synchronized 阻塞会钉住载体线程，优先 ReentrantLock。

## 原稿精华要点

### 核心概念速记

- **进程 vs 线程**：进程是资源分配单位（独立内存）；线程是 CPU 调度单位，同进程线程共享堆内存。
- **并发 vs 并行**：并发是交替推进（单核也能）；并行是同一时刻真在跑（需多核）。
- **竞态条件**：结果取决于线程时序的 bug 总根源；临界区是访问共享资源的代码片段，同一时刻只许一个线程进。
- **原子性/可见性/有序性**：`i++` 非原子（读-加-写三步）；线程各有缓存，修改可能别人看不见；编译器与 CPU 会重排指令，单线程无感、多线程翻车。

### 线程生命周期与基本功

- 6 状态：NEW → RUNNABLE → BLOCKED（等锁）/ WAITING / TIMED_WAITING → TERMINATED。
- 必须调 `start()` 才有新线程，直接调 `run()` 只是普通方法；重复 `start()` 抛异常。
- `sleep` 不释放锁，`wait` 释放锁（且必须在 synchronized 里、配 notify 使用）。
- 中断是协作式的：`interrupt()` 只置标志并让阻塞方法抛 `InterruptedException`，catch 后要么恢复标志要么退出，吞掉等于线程停不下来。
- 用普通 boolean 做跨线程停止开关必翻车（可见性），开关要 volatile 或用中断。

### JUC 工具一句话地图

| 工具 | 一句话 | 细节篇 |
|---|---|---|
| ReentrantLock | 可重入锁，多 tryLock / 公平 / 多 Condition | [synchronized与锁](开发语言/Java/并发/synchronized与锁.md) |
| CountDownLatch | 等 N 件事全做完，一次性 | [AQS](开发语言/Java/并发/AQS.md) |
| CyclicBarrier | N 个线程在栅栏会合再一起走，可循环 | [AQS](开发语言/Java/并发/AQS.md) |
| Semaphore | 许可限流，允许 N 个并发 | [AQS](开发语言/Java/并发/AQS.md) |
| ReadWriteLock | 读共享写独占 | [synchronized与锁](开发语言/Java/并发/synchronized与锁.md) |
| CopyOnWriteArrayList | 读多写极少，写时复制 | 本篇容器一句话，深入看集合方向 |
| BlockingQueue | 生产者消费者解耦 | [线程池](开发语言/Java/并发/线程池.md)（队列选型） |

### 三条生产红线

- 禁止无界队列的快捷线程池工厂，参数手写、线程命名、拒绝策略按业务选。
- ThreadLocal 凡 set 必 finally remove，尤其在线程池环境。
- 死锁先 `jstack` 留现场再动手，预防靠固定加锁顺序与带超时的 tryLock。

## 异步编排的位置

`CompletableFuture` 的编排 API（thenApply/thenCompose/thenCombine/exceptionally）与 Spring `@Async` 的用法在 [Async-异步](开发语言/Java/Async-异步.md) 篇展开；本系列只强调一条：异步任务必须显式指定业务线程池，默认 commonPool 只配跑纯计算短任务。并行调多个下游接口再聚合结果，是它的典型落点。

## 小技巧与冷知识

【小技巧】并发 bug 复现不了时，先把“时序”变成可控变量：单测里用 `CountDownLatch` 让两个线程同时起跑、再用 `CyclicBarrier` 让它们在临界点会合，比“循环跑一万次碰运气”更容易把竞态打出来。原文只给了工具一句话地图，这条是把它用在复现上的具体打法——能稳定复现的竞态才修得动。

【小技巧】`wait()` 必须写在 `while` 循环里判条件，不能只写 `if`：虚假唤醒（spurious wakeup）会让线程在条件没成立时也被唤醒，`if` 判断只走一次就直接往下跑，`while` 会重新检查条件再决定等不等。原文只写了 wait 必须在 synchronized 里、配 notify 用，这条是同一场景里最容易漏的第二条规矩。

【小技巧】线程池里的线程名是排障的第一现场：创建线程池时用自定义 `ThreadFactory` 给线程起带业务前缀的名字（如 `order-query-1`），jstack 里一眼能看出这条栈是哪个池的、正在干什么。默认的 `pool-1-thread-1` 在多个池并存时根本分不清谁是谁，命名是原文“线程命名”那条红线的落地做法。

【小技巧】主线程结束不等于进程结束：只要还有一个非守护线程活着，JVM 就不会退出，后台刷数据的线程没设成守护线程、主流程跑完进程也会一直挂着。线程池的线程默认是非守护的，应用停不下来时先查是不是某个池没 shutdown，而不是怀疑主逻辑卡住了。

【小技巧】`join()` 是最简单的“等它跑完”：主线程调 `worker.join()` 会等 worker 线程结束再继续，比轮询 `isAlive()` 省事，也比 sleep 固定几秒可靠。等多个线程时用 `CountDownLatch` 或 `Future.get()` 更合适，join 只适合一两个线程的简单等待场景。

【小技巧】`CompletableFuture` 不传池就走公共池，公共池的并行度默认是 CPU 核数减 1：原文已强调异步任务要显式指定业务线程池，根子就在这里——公共池满了会拖慢所有用它的任务。IO 密集的编排任务传自己的池，纯计算短任务才留给公共池，别让下游调用堵住公共池里的计算任务。

【小技巧】读多写极少的列表用 `CopyOnWriteArrayList`，但别在写多的地方用它：每次写都复制整个数组，读不加锁、写代价大，监听器列表、配置快照这类“写极少、读极多”的场景才划算。写频繁的列表用它，复制开销会把读的收益全吃掉，原文工具表里只写了“一句话”，选型时先看读写比。

【小技巧】`volatile` 只能保证可见性和有序性，管不了原子性：`volatile int count; count++` 仍然是读-加-写三步，两个线程同时加还是会丢。原文速记里已讲 `i++` 非原子，这条是对应的修复口径——计数场景用 `AtomicInteger`，复合操作（先查后改）用锁，别指望 volatile 一人顶三件事。

【冷知识】Java 的线程和操作系统线程是一一对应的（JDK 21 之前）：`new Thread` 就是向操作系统要一个原生线程，所以线程数上千后内存和调度开销会先撑不住。虚拟线程（JDK 21）改成多对一映射才把这个天花板抬高，原文索引里把它单列一篇、并提醒 synchronized 会钉住载体线程，根子就是这层映射关系变了。

【冷知识】`Thread` 的名字“线程”来自操作系统的线程概念，但 JVM 里的线程状态和操作系统线程状态并不一一对应：Java 的 BLOCKED 是“等 synchronized 锁”，WAITING 是“等 notify/park”，操作系统层面可能都只是“挂起”。jstack 里看到的状态是 JVM 视角的，排查“线程为什么不动”时先按 Java 状态分流，别直接套操作系统的状态解释。

【小技巧】中断标志被吞掉后线程就停不下来，恢复标志只要一行：`catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }`，先恢复中断标志、再退出。原文已讲 catch 后要么恢复标志要么退出，这行代码就是“恢复标志”的标准写法——只记日志不恢复，循环上层再也收不到中断信号。

【冷知识】`synchronized` 在 JDK 6 之前是重量级锁，之后才有了偏向锁、轻量级锁、自适应自旋这套升级路径：无竞争时几乎不进内核，有竞争才逐步升级。原文把 synchronized 的分工放在子篇，这条是它“性能不如从前那么差”的历史背景——早期“能不用 synchronized 就不用”的说法，在 JDK 6 之后已经过时了。

## 面试与自测

1. 用普通 boolean 做跨线程停止开关为什么必翻车？正确做法是什么？
2. `sleep` 和 `wait` 在锁方面的区别是什么？`wait` 的使用有什么前提？
3. 本篇三条生产红线中，关于线程池和 ThreadLocal 的要求各是什么？

### 参考答案

1. 必翻车在可见性上：线程各有缓存，普通 boolean 的修改可能别的线程看不见。开关要声明为 volatile，或用中断机制。
2. `sleep` 不释放锁，`wait` 释放锁；`wait` 必须在 synchronized 里使用，并配 notify 使用。中断是协作式的：`interrupt()` 只置标志并让阻塞方法抛 `InterruptedException`，catch 后要么恢复标志要么退出，吞掉等于线程停不下来。
3. 线程池：禁止无界队列的快捷线程池工厂，参数手写、线程命名、拒绝策略按业务选。ThreadLocal：凡 set 必 finally remove，尤其在线程池环境，否则会串数据与泄漏。

## 关联知识

- 操作系统视角的进程与线程：[操作系统-进程线程](计算机基础/操作系统-进程线程.md)
- 运行时内存区域（与 JMM 区分）：[JVM-内存模型](开发语言/Java/JVM/JVM-内存模型.md)
- DCL 单例的写法落点：[单例](开发语言/Java/设计模式/单例.md)
