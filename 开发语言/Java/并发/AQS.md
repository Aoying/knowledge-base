---
知识点: AQS
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-11-08
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

## 8. 排队、唤醒与公平性定量

本节所有数字均为手算，Python 3 本地复算；【官方】表示 JDK 官方文档 / JDK 源码注释 /《Java 并发编程实战》中的事实，【估算】表示在简化假设下的手算，不是官方数字、不是实测值。凡未标【官方】的数字一律按【估算】理解。

**版本口径（重要）**：本节的 waitStatus 取值与入队/唤醒步骤名是经典版 AQS 的实现（JDK 8–17，与本文原文小技巧节一致）。JDK 21 起 AQS 已重写：`Node.waitStatus` 被 `status` 位字段取代（`WAITING=1`、`COND=2`、`CANCELLED=0x80000000`），`SIGNAL`/`CONDITION`/`PROPAGATE` 常量与 `shouldParkAfterFailedAcquire`/`unparkSuccessor` 方法名不再存在；17u/21u 的后续更新也已合入重写（17.0.12 已合入，2026-11-08 实取 jdk17u 源码核验）。排队、park/unpark、公平性结论不受影响，变的是内部字段与方法名；读 JDK 21+ 或最新 17u/21u 源码时以 `status` 位字段为准。

### 8.1 CLH 队列节点状态机：四个值怎么流转

【官方】`Node.waitStatus` 只有五个取值：`0`（初始，无特殊含义）、`CANCELLED(1)`、`SIGNAL(-1)`、`CONDITION(-2)`、`PROPAGATE(-3)`。注意状态描述的是“后继节点需要什么”，不是本节点自己的状态：一个节点置 `SIGNAL`，意思是“我释放或取消时必须 unpark 我的后继”。

| 值 | 含义 | 怎么进去 | 怎么出来 |
|---|---|---|---|
| `0` | 初始 / 无待办 | 节点新建时就是 `0` | 后继节点把它 CAS 成 `SIGNAL`；条件队列转回同步队列时也回到 `0` |
| `SIGNAL(-1)` | 后继已 park，等我唤醒 | 后继节点入队后，把前驱的 `waitStatus` 从 `0` CAS 成 `SIGNAL` | 释放时 `unparkSuccessor` 把它 CAS 回 `0`，再 unpark 后继 |
| `CANCELLED(1)` | 本节点线程已取消（超时 / 中断），终态，不再变回去 | 超时或中断时把自己的 `waitStatus` 置 `CANCELLED` | 不出来；出队时由后继 / 前驱绕过它，`CANCELLED` 节点本身不再参与唤醒链 |
| `CONDITION(-2)` | 在条件队列里，不在同步队列里 | `Condition.await()` 入条件队列时置 `CONDITION` | `signal()` 转入同步队列时 CAS 成 `0`，再按同步队列规则排队 |
| `PROPAGATE(-3)` | 共享模式下释放要继续向后传播 | 共享释放 `doReleaseShared` 时（仅头节点）置 `PROPAGATE` | 后继共享节点获得后继续传播，或下一轮释放覆盖 |

三条流转铁律（【官方】，见 JDK 源码注释）：第一，`CANCELLED` 是终态，节点一旦取消就永不回头，取消的线程也“永不再阻塞”；第二，`CONDITION` 节点在转回同步队列之前不参与同步队列的唤醒，`signal()` 必须先把它 CAS 成 `0` 再入队；第三，`PROPAGATE` 只出现在共享模式，独占释放不走这条路——读锁能成批放行、写锁一次只唤醒一个，根子就在这里。

### 8.2 一段 N 线程排队的入队 / 唤醒步骤计数【估算】

简化模型（只数命名的 CAS 操作与 park/unpark，`setHead`、链表指针赋值等普通写不计入；无取消、无中断、无超时）：设 1 个持有者 + `N = 8` 个排队线程，每个等待者恰好获得一次锁、再释放。

入队（每个等待者都一样）：

1. `tryAcquire` 的 state CAS 失败 1 次（锁正被占着，`0 → 1` 抢不到）；
2. `addWaiter` 的 tail CAS 成功 1 次（把自己挂到队尾）；
3. `shouldParkAfterFailedAcquire` 把前驱 `waitStatus` 从 `0` CAS 成 `SIGNAL`，1 次；
4. `park()` 挂起 1 次。

所以入队阶段：CAS `8 × 3 = 24` 次，park `8` 次。

交接（8 个等待者各获得一次锁，共 8 轮“释放 → 唤醒 → 获得”，外加最后一个等待者最后一次释放时队列已空）：每轮释放者做 state CAS（`1 → 0`）1 次、把头节点 `waitStatus` 从 `SIGNAL` CAS 回 `0` 共 1 次、新持有者 `tryAcquire` 的 state CAS 成功 1 次、unpark 后继 1 次；最后一轮只有 state CAS 1 次、无 unpark。

所以交接阶段：CAS `8 × 3 + 1 = 25` 次，unpark `8` 次。

全程合计：CAS `24 + 25 = 49` 次，park + unpark `8 + 8 = 16` 次。换句话说，8 个人排队走完一轮，每人平均摊到约 `49 ÷ 8 ≈ 6.1` 次 CAS 和 `2` 次 park/unpark——排队本身的 CAS 开销是 `O(N)` 量级、与人数成正比，而不是 `O(N²)`；真正的成本大头不在 CAS 次数，而在每次 park/unpark 背后的线程切换（见 8.3）。

### 8.3 公平锁 vs 非公平锁：插队条件、饥饿上界与吞吐换算

插队成功的条件（【官方】）：非公平锁的后来者在入队前先直接 `tryAcquire` CAS 抢一次，只有在抢的那一刻 `state == 0`（锁恰好空闲）且 CAS 成功时才算插队成功；公平锁跳过这一步，入队前先查队列里是否有人排队，有人就自觉排队。同一条规则的另一面：无参 `tryLock()` 不遵守公平性设置，公平锁上它也能插队（【官方】，`ReentrantLock` 类注释）。

饥饿上界【估算】：设排在某线程前面的人数为 `Q`，临界区时长为 `T_cs`，忽略唤醒开销，则公平锁下该线程最坏等待时间 `W_fair = Q × T_cs`——前面每个人最多占一个临界区，轮到它之前不会再有人插到它前面，所以上界存在且是线性的。非公平锁没有这个上界：只要不断有新线程在锁空闲的瞬间抢到，队首线程可以一直等下去，最坏等待时间无上界（【官方】定性结论：公平锁“保证不饥饿”，非公平锁“不保证任何特定的访问顺序”）。

| 场景（【估算】） | `Q`（前面人数） | `T_cs`（临界区） | 公平锁最坏等待 `Q × T_cs` | 非公平锁最坏等待 |
|---|---|---|---|---|
| 短临界区、中等排队 | 7 | 10 µs | 70 µs | 无上界 |
| 短临界区、短排队 | 7 | 1 µs | 7 µs | 无上界 |
| 短临界区、长排队 | 15 | 10 µs | 150 µs | 无上界 |
| 长临界区、短排队 | 3 | 50 µs | 150 µs | 无上界 |

吞吐差异的方向与量级【估算】：公平锁每次交接都要经过“释放者 unpark → 被唤醒线程从 park 中恢复 → 线程切换 → 重新抢锁”这一整套流程，记这次切换成本为 `C_sw`；非公平锁插队命中时直接 CAS 拿到锁，跳过了唤醒与切换。取假设值 `C_sw = 5 µs`（不是实测值，只是换算用的假设量），则每轮锁交接的周期为：公平锁 `T_cs + C_sw`，非公平锁（插队命中）`T_cs`，吞吐比（非公平 / 公平）`= (T_cs + C_sw) / T_cs`。

| `T_cs`（临界区） | 公平锁每轮周期 `T_cs + 5 µs` | 非公平锁每轮周期（插队命中） | 吞吐比 |
|---|---|---|---|
| 1 µs | 6 µs | 1 µs | 约 6.0 倍 |
| 10 µs | 15 µs | 10 µs | 约 1.5 倍 |
| 50 µs | 55 µs | 50 µs | 约 1.1 倍 |

结论只看方向，不看具体倍数：临界区越短，线程切换成本占比越高，公平锁的吞吐惩罚越大；临界区越长，两者差距越小。这与【官方】定性结论一致——`ReentrantLock` 类注释写明，公平锁在多线程竞争下“整体吞吐更低（往往低得多），但获得锁的时间方差更小、保证不饥饿”。所以默认用非公平锁不是“懒”，而是临界区通常很短时、唤醒成本主导了总开销；一旦业务要求等待时间有上界（不能饿死），才值得付这笔切换成本换公平。

### 8.4 state 的 CAS 竞争：N 个线程同时抢锁

单轮抢锁【估算】：`N` 个线程在同一时刻对空闲锁做 `tryAcquire` 的 state CAS（`0 → 1`），恰好 1 个成功、`N - 1` 个失败，首轮成功率 `1 / N`。失败者不原地自旋重试，而是入队 park（见 8.2），所以单轮 CAS 失败总数就是 `N - 1`，不会像纯自旋锁那样随重试轮数线性膨胀。

| `N`（同时抢锁的线程数） | 首轮成功 | 首轮 CAS 失败 | 首轮成功率 |
|---|---|---|---|
| 4 | 1 | 3 | 25.0% |
| 8 | 1 | 7 | 12.5% |
| 16 | 1 | 15 | 6.25% |
| 32 | 1 | 31 | 3.125% |

自旋 vs 阻塞 park 的切换阈值思路【估算】：自旋一次迭代的成本记为 `c`（约几十纳秒量级的假设），park + unpark 一整套的成本记为 `C_park`（微秒量级的假设），盈亏平衡的自旋次数 `k* = C_park / c`。取 `c = 50 ns`：`C_park = 2 µs` 时 `k* = 40` 次，`C_park = 5 µs` 时 `k* = 100` 次，`C_park = 10 µs` 时 `k* = 200` 次。决策规则是期望等待时间小于 `k* × c` 就自旋等，大于就 park——等的时间越短越值得自旋，等的时间越长越该 park 让出 CPU。AQS 的选择与这条规则一致：`acquireQueued` 里线程只在“前驱是头节点且可能马上拿到锁”的窗口内短暂重试，确认拿不到就置 `SIGNAL` 后 park，而不是无上限自旋；`tryLock(timeout)` 的超时等待同样走入队 park，而不是空转烧 CPU。

### 8.5 边界条件

1. `N = 1`（无人排队）时没有队列、没有 park/unpark：后来者要么一次 CAS 直接拿到锁，要么一次 CAS 失败后建队——8.2 的计数里“每人 3 次 CAS + 1 次 park”只在“已有人排队”的前提下成立，单线程场景下排队开销为零，别把排队模型套到无竞争场景上。
2. 队首线程被取消时，释放者 unpark 的不是它：`unparkSuccessor` 发现后继节点是 `CANCELLED`，会从队尾向前扫描、跳过所有已取消节点，唤醒第一个未取消的后继；若整条队列只剩取消节点，本次释放不 unpark 任何人——取消不会“堵住”唤醒链，但会让释放者多做一次从尾到头的扫描。
3. 头节点 `waitStatus == 0` 时释放不 unpark：`unparkSuccessor` 只有在头节点是 `SIGNAL` 时才 unpark 后继；头节点为 `0` 说明后继尚未 park（还没来得及置 `SIGNAL`）或根本没有后继，这时不唤醒是正确的——后继线程 park 之前会再检查一次前驱与 state，不会把唤醒信号丢在“释放已发生、park 还没执行”的窗口里（park/unpark 的许可语义兜底）。
4. `PROPAGATE` 只在共享模式出现，独占锁永远不会置它：读锁释放走 `doReleaseShared` 传播唤醒、写锁释放只唤醒下一个节点；把独占锁的释放按“会连锁唤醒一批”去理解，或把共享释放按“只唤醒一个”去调优，都是错的。
5. 公平性的边界在 `tryLock()` 处断开：无参 `tryLock()` 不遵守公平性设置，在公平锁上也能插队成功；只有 `lock()` 与带超时的 `tryLock(timeout)` 才走公平排队。把“用了公平锁”当成“任何入口都公平”，会在 `tryLock()` 的调用点上误判。

### 复算口径

```python
# Python 3 本地复算（手算复核，2026-11-08）
# 8.2: 1 持有者 + N=8 排队线程, 每人获得一次锁再释放
N = 8
enq_cas = N * 3          # 每人: tryAcquire 失败 CAS 1 + tail CAS 1 + 前驱 SIGNAL CAS 1
enq_park = N             # 每人 park 1 次
handover_cas = 8 * 3 + 1 # 8 轮交接每轮 3 次 CAS + 最后一轮释放 1 次
handover_unpark = 8      # 8 轮每轮 unpark 1 次
print(enq_cas, enq_park)                 # 24 8
print(handover_cas, handover_unpark)     # 25 8
print(enq_cas + handover_cas)            # 49
print(enq_park + handover_unpark)        # 16
print(round((enq_cas + handover_cas) / N, 1))  # 人均约 6.1 次 CAS

# 8.3: 公平锁最坏等待 W = Q * T_cs (us)
for Q, Tcs in [(7, 10), (7, 1), (15, 10), (3, 50)]:
    print(Q, Tcs, Q * Tcs)               # 70 7 150 150

# 8.3: 吞吐换算, 假设切换成本 C_sw = 5 us
C_sw = 5
for Tcs in [1, 10, 50]:
    print(Tcs, Tcs + C_sw, round((Tcs + C_sw) / Tcs, 2))  # 6/1.5/1.1 倍

# 8.4: N 线程同时抢锁, 首轮失败 N-1
for N in [4, 8, 16, 32]:
    print(N, N - 1, round(1 / N, 3))    # 3/7/15/31; 0.25/0.125/0.062/0.031

# 8.4: 自旋/park 盈亏次数 k* = C_park / c, c = 50 ns
for C_park_us in [2, 5, 10]:
    print(C_park_us, int(C_park_us * 1000 / 50))  # 40 100 200
```

来源（核验日期 2026-11-08）：

- 【官方】JDK 17 `ReentrantLock` Javadoc（公平锁吞吐更低、保证不饥饿、无参 `tryLock()` 不遵守公平性）：https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html（主代理实读核验）
- 【官方】OpenJDK JDK 8u `AbstractQueuedSynchronizer` 源码（经典版 `waitStatus` 取值 `CANCELLED=1`/`SIGNAL=-1`/`CONDITION=-2`/`PROPAGATE=-3`）：https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java（主代理实取核验）
- 【官方·版本注】OpenJDK jdk17u master `AbstractQueuedSynchronizer` 源码（重写后：`Node.status` 位字段，`WAITING=1`、`COND=2`、`CANCELLED=0x80000000`，无 `waitStatus`）：https://github.com/openjdk/jdk17u/blob/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java（主代理实取核验；同款重写亦见于 JDK 21 GA 标签源码）
- 其余数字均为【估算】：8.2 的步骤计数用简化模型（只数命名的 CAS 与 park/unpark）、8.3 的 `C_sw = 5 µs` 与 8.4 的 `c = 50 ns` 均为换算用假设值，不是官方数字、不是实测值。

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

## 9. 面试与自测

- 一句话讲清 AQS：state + 队列分别管什么？
- ReentrantLock 的公平与非公平差别在代码上落在哪一步？
- Semaphore 与 CountDownLatch 的 state 语义各是什么？

- 8 个线程排队等一把非公平锁：入队与交接全程一共多少次 CAS、多少次 park/unpark？公平锁下排第 7 的线程最坏要等多久（用排队人数 × 临界区时长表示）？

### 参考答案

1. state 是同步状态整数，含义由子类定义：独占模式下 0 表示空闲、>0 表示重入次数，共享模式下表示剩余许可数或剩余计数；队列是 CLH 变体的 FIFO 等待队列，负责排队、阻塞（park）和唤醒（unpark）。一句话：state 管“能不能过”，队列管“过不了时怎么等”。
2. 差别只在入队前一步：非公平锁 tryAcquire 先直接 CAS 抢一把，抢不到再入队，后来者可以插队，这就是“非公平”的来源；公平锁入队前先检查队列里是否有人排队，有人就自觉排队，不直接抢。入队之后的排队、阻塞、唤醒流程两者相同。
3. Semaphore 的 state 是剩余许可数：acquire 拿许可，没有许可就排队，release 还许可，是限流的标准件。CountDownLatch 的 state 是需要等待的事件数：countDown 减一，减到 0 放行所有等待者，且一次性、不可重置；要循环多轮任务会合用 CyclicBarrier。

4. 简化模型下（只数命名的 CAS 与 park/unpark，无取消、无中断、无超时）：入队阶段每个等待者 3 次 CAS（tryAcquire 失败 1 次、tail CAS 1 次、前驱 SIGNAL CAS 1 次）+ park 1 次，8 人共 CAS 24 次、park 8 次；交接阶段 8 轮每轮 3 次 CAS（释放 state CAS、清 SIGNAL CAS、新持有者 acquire CAS）+ 最后一轮释放 1 次，共 CAS 25 次、unpark 8 次；全程合计 CAS 49 次、park + unpark 16 次，人均约 6.1 次 CAS。公平锁下排第 7 的线程前面有 7 人，最坏等待 `7 × T_cs`（临界区时长 `T_cs = 10 µs` 时为 70 µs）；非公平锁没有这个上界，不断有新线程插队成功时队首线程可以一直等下去。

## 10. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 使用方视角：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- CAS 基础：[CAS与原子类](开发语言/Java/并发/CAS与原子类.md)
- 分布式计数落地：[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)
