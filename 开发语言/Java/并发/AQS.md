---
知识点: AQS
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-23
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

## 8. 面试与自测

- 一句话讲清 AQS：state + 队列分别管什么？
- ReentrantLock 的公平与非公平差别在代码上落在哪一步？
- Semaphore 与 CountDownLatch 的 state 语义各是什么？

## 9. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 使用方视角：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- CAS 基础：[CAS与原子类](开发语言/Java/并发/CAS与原子类.md)
- 分布式计数落地：[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)
