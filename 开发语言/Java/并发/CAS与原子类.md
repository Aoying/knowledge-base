---
知识点: CAS与原子类
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-23
关联:
  - Java-并发编程
  - AQS
  - ConcurrentHashMap
标签:
  - Java
  - 并发
  - 原理
---

# CAS 与原子类

## 1. 核心概念

- CAS（Compare And Swap）：一条 CPU 指令完成「比较内存里的值是否等于预期值，相等才替换成新值」的原子动作，是无锁编程的基石。与 [synchronized与锁](开发语言/Java/并发/synchronized与锁.md) 的悲观路线相对，CAS 是乐观路线：先假设没冲突，直接改，改不动再重试。
- Java 层入口是 `sun.misc.Unsafe`（JDK 9+ 官方替代为 `VarHandle`），业务代码不直接碰它，用 `java.util.concurrent.atomic` 包的原子类即可。
- 适用边界：竞争越激烈，CAS 自旋重试越浪费 CPU；低中竞争、操作短小（计数、状态位、引用替换）时它赢，高竞争或临界区稍长时锁反而更划算。

## 2. 原子类家族

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();                 // 原子自增并返回新值
count.compareAndSet(1, 2);               // 期望是 1 才改成 2，返回是否成功

AtomicReference<State> ref = new AtomicReference<>(State.INIT);
ref.compareAndSet(State.INIT, State.RUNNING);   // 无锁状态机流转

AtomicIntegerFieldUpdater<Job> updater =
    AtomicIntegerFieldUpdater.newUpdater(Job.class, "retries");  // 给已有类的 volatile 字段加原子更新能力
```

| 类别 | 代表 | 用途 |
|---|---|---|
| 基本类型 | AtomicInteger / AtomicLong / AtomicBoolean | 计数、标志、序号 |
| 引用类型 | AtomicReference / AtomicStampedReference | 无锁替换对象、带版本号防 ABA |
| 数组 | AtomicIntegerArray 等 | 按下标原子更新 |
| 字段更新器 | AtomicXxxFieldUpdater | 存量类字段不改类型做原子更新 |
| 累加器 | LongAdder / DoubleAdder | 高并发计数（见第 4 节） |

## 3. CAS 的自旋与 ABA 问题

```java
// incrementAndGet 的等价骨架：自旋重试直到成功
public int incr(AtomicInteger v) {
    int old;
    do {
        old = v.get();
    } while (!v.compareAndSet(old, old + 1));
    return old + 1;
}
```

- **ABA 问题**：线程读到值 A，期间别人把 A 改成 B 又改回 A，本线程 CAS 仍会成功——值没变，但「过程」变了。若值承载了状态历史（如栈顶指针、引用复用），就会误判。对策：`AtomicStampedReference` 给值配一个版本号（邮戳），值与版本一起 CAS，A→B→A 时版本已变，CAS 失败。
- **自旋成本**：失败重试烧 CPU。缓解手段：重试加退避（短暂让步/随机延迟）、限制重试次数后降级为加锁、或直接选对工具（高并发计数换 LongAdder）。

## 4. LongAdder：高并发计数的正解

- 痛点：`AtomicLong` 高并发下所有线程 CAS 同一个变量，缓存行反复失效，大部分尝试都在失败重试。
- 做法：把计数分散到多个 `Cell` 上（按线程哈希落到不同 Cell，Cell 之间填充隔离避免伪共享），各自 CAS 累加；`sum()` 时再把 base 与所有 Cell 加总。
- 取舍：`sum()` 不是强一致快照（读取瞬间可能还有更新在途），它换的是写入吞吐。统计 QPS、累计金额笔数、埋点计数这类「只增、容忍读到近似值」的场景用它；需要精确读-改-写的场景不要用。
- 落点：限流计数、接口调用统计、批处理进度；`ConcurrentHashMap` 的计数与扩容协助也用了同样的分段思想，见 [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)。

## 5. 实战应用

```java
// 无锁状态流转：任务状态只允许 INIT -> RUNNING -> DONE，CAS 保证只有一个线程能推进
class Job {
    enum S { INIT, RUNNING, DONE }
    private final AtomicReference<S> state = new AtomicReference<>(S.INIT);
    boolean start() { return state.compareAndSet(S.INIT, S.RUNNING); }  // 抢到才执行，天然防重复提交
    void finish()   { state.compareAndSet(S.RUNNING, S.DONE); }
}
```

- **防重复提交**：审批单处理入口先 CAS 抢占状态，抢不到直接返回「处理中」，比先查后改的 check-then-act 少一个竞态窗口（跨实例仍需数据库/分布式手段兜底）。
- 与锁的组合：现实代码常是「先 CAS 快路径，冲突严重时落锁」；`ReentrantLock` 与 [AQS](开发语言/Java/并发/AQS.md) 内部抢 state 也是 CAS。CAS 不是锁的替代品，而是锁的实现材料之一。
- JDK 9+ 新代码了解 `VarHandle` 即可：它提供与 Unsafe 等价的原子与内存序操作且类型安全，框架层已逐步迁移，业务层继续用原子类。

## 6. 常见误区

- 以为 CAS 成功就代表「这段时间没人动过」：ABA 说明值相等不等于没变过，敏感场景带版本。
- 用原子类拼复合操作：`if (ref.get() == X) ref.set(Y)` 两步之间仍有竞态，必须 `compareAndSet` 一步到位；多个原子变量各自原子，不等于组合起来原子（如同时维护 count 与 sum）。
- 高并发计数硬上 `AtomicLong`：能跑，但吞吐被单个热点缓存行锁死，该换 LongAdder。
- 在 CAS 循环里做重活：每次重试都会重做一遍，循环体只放「读旧值、算新值」这种纯计算，重活移出去。

## 7. 面试与自测

- 手写一个基于 CAS 的自增，并说明失败重试的成本从哪来。
- ABA 是什么？发生在哪些数据结构上最危险？两种解法？
- AtomicLong 与 LongAdder 的适用场景如何分界？

## 8. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 悲观路线对照：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- 分段思想应用：[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)
- AQS 里的 CAS：[AQS](开发语言/Java/并发/AQS.md)
