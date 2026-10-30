---
知识点: AQS
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java-并发编程
  - synchronized与锁
  - CAS与原子类
标签:
  - Java
  - 并发
  - 原理
---

# AQS

## 1. 核心概念

- AQS（`AbstractQueuedSynchronizer`）是 JUC 的同步器框架：ReentrantLock、Semaphore、CountDownLatch、ReentrantReadWriteLock 的等待队列，乃至 `FutureTask`，底层都基于它。理解 AQS 等于拿到一把读所有 JUC 同步工具源码的钥匙。
- 它只做两件事：**管理一个状态整数 `state`** + **维护一个 FIFO 等待队列**。具体同步器只定义「state 怎么变才算拿到/放行」，排队、阻塞、唤醒的脏活全由 AQS 承包。

## 2. 核心结构

```java
// AQS 的骨架（简化示意）
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;                       // 同步状态：含义由子类定义
    // 一条双向链表（CLH 变体）挂等待线程，每个节点包一个 Thread

    protected boolean tryAcquire(int arg) { throw new UnsupportedOperationException(); }
    protected boolean tryRelease(int arg) { throw new UnsupportedOperationException(); }
    protected int tryAcquireShared(int arg) { throw new UnsupportedOperationException(); }
    protected boolean tryReleaseShared(int arg) { throw new UnsupportedOperationException(); }
}
```

- `state` 用 volatile 保证可见性，修改一律走 [CAS](开发语言/Java/并发/CAS与原子类.md)：抢锁本质就是 CAS 把 state 从 0 改成 1（或累加计数）。
- 等待队列是 CLH 队列的变体：节点自旋/阻塞等待前驱节点状态变化，而不是所有线程盯着同一个变量抢，避免惊群式的无效竞争。
- 抢不到的线程：封装成 Node 入队尾，然后 `LockSupport.park()` 挂起（不占 CPU）；释放者改完 state 后 `unpark` 唤醒队列里下一个符合条件的节点。

## 3. 独占与共享两种模式

| 模式 | 语义 | 代表实现 | state 的含义 |
|---|---|---|---|
| 独占 | 同一时刻只有一个线程持有 | ReentrantLock、写锁 | 0 空闲 / >0 为重入次数 |
| 共享 | 多个线程可同时通过 | Semaphore、CountDownLatch、读锁 | 剩余许可数 / 剩余计数 |

以 ReentrantLock 为例看 state 的复用：非公平锁 `tryAcquire` 先 CAS 抢一把（后来者可以插队，这是「非公平」的来源），抢不到再走 AQS 入队；重入 = 同一线程重复 acquire，state 递增，释放时递减到 0 才真正放锁。公平锁只多一步：入队前先检查队列里有没有人排队，有人就自觉排队。

## 4. 基于 AQS 的工具速览

```java
// CountDownLatch：state = 需要等待的事件数；countDown 减一，减到 0 放行所有等待者（一次性，不可重置）
CountDownLatch latch = new CountDownLatch(3);
latch.countDown();          // 某件事做完
latch.await();              // 等 3 件事全做完

// Semaphore：state = 许可数；acquire 拿许可（没有就排队），release 还许可 —— 限流的标准件
Semaphore sem = new Semaphore(10);
sem.acquire();
try { callDownstream(); } finally { sem.release(); }
```

- `CyclicBarrier` 是个例外：它基于 ReentrantLock + Condition 实现，不是直接继承 AQS；它可循环复用，与 Latch 的一次性语义区分开。
- `ReentrantReadWriteLock`：一个同步器里塞两个状态——state 高 16 位记读锁计数、低 16 位记写锁重入数，读写互斥、读读共享。

## 5. 自定义同步器（理解深度的检验）

```java
// 不可重入的独占锁：state 0/1，几十行就能写完 —— 写过一遍，AQS 就不再是黑盒
class Mutex extends AbstractQueuedSynchronizer {
    @Override protected boolean tryAcquire(int arg) {
        return compareAndSetState(0, 1);           // 抢到就置 1，并记录持有者（略 setExclusiveOwnerThread）
    }
    @Override protected boolean tryRelease(int arg) {
        return compareAndSetState(1, 0);           // 演示用：生产写法必须先校验持有者是当前线程
    }
    @Override protected boolean isHeldExclusively() { return getState() == 1; }
    Condition newCondition() { return new ConditionObject(); }
}
```

自定义同步器只需重写 try 系方法并选用独占/共享入口（`acquire` / `acquireShared`），排队与阻塞唤醒由模板方法兜底——这正是模板方法模式在 JDK 源码里的教科书级落点。

## 6. 原理剖析要点

- **模板方法骨架**：`acquire` 的流程是「tryAcquire 失败 → 建 Node 入队 → 循环检查前驱是不是头节点且能 tryAcquire → park 挂起 → 被唤醒后重试，头节点成功后出队接任」。读源码抓这条主线即可，其余是中断处理与节点状态的细节。
- **为什么被唤醒后要看前驱**：只有前驱是头节点（即自己排在最前）才有资格尝试获取，保证 FIFO 公平性骨架；非公平锁是在入队之前多给一次直接抢的机会。
- **取消与超时**：`acquire` 的可中断、带超时变体在 AQS 内统一处理节点取消（标记 CANCELLED 并从队列摘除），这也是 `tryLock(timeout)` 能实现的原因，见 [synchronized与锁](开发语言/Java/并发/synchronized与锁.md)。

## 7. 常见误区

- 把 AQS 当业务工具直接用：业务代码用它暴露出来的成品（锁、闩、信号量），继承 AQS 只在写框架/中间件时才需要。
- 以为公平锁「更公平所以更好」：公平意味着每次唤醒都要严格按序、线程切换更频繁，吞吐明显低于非公平，默认别开。
- 混淆 Latch 与 Barrier 的可复用性：Latch 计数到 0 后不可重置，要循环等待多轮任务会合用 Barrier。

## 小技巧与冷知识

【小技巧】`tryLock()` 没排队、没 park，直接试一次 CAS 就返回：`ReentrantLock.tryLock()` 底层只调一次 `tryAcquire`，抢不到立刻返回 false，不会进等待队列。想“抢不到就干别的、过会儿再试”的降级逻辑用它；把它当成“带超时的 tryLock”会误判——带超时的 `tryLock(timeout)` 才会入队等唤醒，两者走的根本不是一条路。

【小技巧】看懂等待队列先认四个节点状态：`Node` 的 waitStatus 只有四个值——`CANCELLED(1)` 已取消、`SIGNAL(-1)` 后继节点要等我唤醒、`CONDITION(-2)` 在条件队列里、`PROPAGATE(-3)` 共享模式下要传播唤醒。读 AQS 源码卡住时，先把这四个值对上号，`acquireQueued`、`unparkSuccessor` 里的分支立刻能读下去；状态是负数是“还在等”，正数是“已取消”，符号本身就有含义。

【小技巧】`getQueueLength()`、`getQueuedThreads()` 返回的只是估计值：队列在并发下随时在变，AQS 文档明确写了这些方法返回的是估计值，别拿它做“当前还有几个人在等锁”的精确判断。排障时用它看趋势可以，做限流、做“没人排队才执行”的业务判断不行，精确的排队感知要靠业务自己的计数器。

【小技巧】共享模式的唤醒会“连锁传播”，独占不会：读锁释放时 `setHeadAndPropagate` 会顺着队列把后面符合条件的读线程一个个唤醒（PROPAGATE 状态就是干这个的），所以读锁能成批放行；写锁释放只唤醒下一个节点。读多写少的场景下“一次释放放行一批读线程”是正常的，不是惊群，别按独占锁的直觉去调它。

【小技巧】`Condition.await()` 会把锁完全释放、再原样拿回来：`await()` 内部先记下当前 state（重入次数），全部释放后才 park，`signal()` 后重新排队、按原重入次数把锁拿满才返回。重入 3 次的线程 await 后，别人能拿到锁，signal 唤醒后它要重新拿到 3 次重入才继续跑；只释放一次的直觉是错的，条件等待和锁的“借出借回”是成套的。

【小技巧】`LockSupport.park()` 不会丢唤醒信号：park/unpark 是“许可”语义——先 `unpark` 再 `park`，park 会立刻返回而不是一直挂起，信号不会丢。这是 AQS 敢用 park/unpark 做阻塞唤醒的根子；`Object.wait()` 没有这层保障，必须在循环里检查条件。自己写同步器时，用 park/unpark 就别再套一层“唤醒丢失”的防御逻辑，重复检查条件才是正道。

【小技巧】公平锁的“公平”只查有没有人排队，不保证绝对按序拿到：公平锁入队前调 `hasQueuedPredecessors()`，队列非空就自觉排队，但 park 唤醒的时机、线程调度的顺序仍可能让后到的线程先跑。公平锁消灭的是“后来者直接插队抢锁”，不是“严格按到达顺序执行”；需要严格顺序的场景要靠业务自己排队，别指望公平锁兜底。

【小技巧】读锁有个 65535 的上限：`ReentrantReadWriteLock` 的 state 高 16 位记读锁计数，读锁重入次数超过 65535 会抛 `Error: Maximum lock count exceeded`。正常业务到不了这个数，但递归里反复加读锁、循环里漏了 `unlock()` 时，读计数是会悄悄往上涨的；看到这个 Error 先查读锁的 unlock 是不是成对的，而不是去调锁的参数。

【冷知识】AQS 出自 Doug Lea 2004 年的论文《The java.util.concurrent Synchronizer Framework》：ReentrantLock、Semaphore、CountDownLatch 这些工具都是这篇论文里的“同一套框架的不同用法”。读源码前先看论文的图——state + CLH 队列 + 模板方法三件套，整篇论文其实就讲了这一件事，JUC 的同步工具全是它的实例化。

【冷知识】CLH 队列的原始论文比 AQS 早十多年：CLH 锁由 Craig、Landin、Hagersten 在 1993–1994 年提出，原始设计里每个线程自旋等前驱节点的标志位；AQS 改成“前驱节点状态变化 + park 挂起”，把自旋换成了阻塞，才适合 Java 这种“线程多、不能全自旋”的环境。理解这层演变，就明白为什么 AQS 叫“CLH 变体”而不是“CLH 锁”。

【冷知识】`FutureTask` 也用 AQS，但 state 含义完全不同：它的 state 是任务状态机（NEW → COMPLETING → NORMAL/EXCEPTIONAL/CANCELLED），`get()` 就是一次共享模式的 acquire。同一个框架、同一套排队唤醒代码，换个 state 语义就长出一个完全不同的工具——这正是原文说“具体同步器只定义 state 怎么变”的意思，AQS 的复用粒度就是 state 的定义。

## 8. 面试与自测

- 一句话讲清 AQS：state + 队列分别管什么？
- ReentrantLock 的公平与非公平差别在代码上落在哪一步？
- Semaphore 与 CountDownLatch 的 state 语义各是什么？

### 参考答案

1. state 是同步状态整数，含义由子类定义：独占模式下 0 表示空闲、>0 表示重入次数，共享模式下表示剩余许可数或剩余计数；队列是 CLH 变体的 FIFO 等待队列，负责排队、阻塞（park）和唤醒（unpark）。一句话：state 管“能不能过”，队列管“过不了时怎么等”。
2. 差别只在入队前一步：非公平锁 tryAcquire 先直接 CAS 抢一把，抢不到再入队，后来者可以插队，这就是“非公平”的来源；公平锁入队前先检查队列里是否有人排队，有人就自觉排队，不直接抢。入队之后的排队、阻塞、唤醒流程两者相同。
3. Semaphore 的 state 是剩余许可数：acquire 拿许可，没有许可就排队，release 还许可，是限流的标准件。CountDownLatch 的 state 是需要等待的事件数：countDown 减一，减到 0 放行所有等待者，且一次性、不可重置；要循环多轮任务会合用 CyclicBarrier。

## 9. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 使用方视角：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- CAS 基础：[CAS与原子类](开发语言/Java/并发/CAS与原子类.md)
- 分布式计数落地：[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)
