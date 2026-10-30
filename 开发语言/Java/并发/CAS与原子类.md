---
知识点: CAS与原子类
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】`incrementAndGet` 和 `getAndIncrement` 只差一个返回值，序号用错会差一：前者返回加完的新值，后者返回加之前的旧值。做“从 1 开始编号”的自增序号用 `incrementAndGet`，做“从 0 开始取下标”用 `getAndIncrement`；两个方法混着写，编号要么从 0 起要么从 1 起，全看你调的是哪一个，代码评审时先看返回值再看逻辑。

【小技巧】`compareAndSet` 的 boolean 返回值必须判：它只告诉你“这次换没换成”，换不成不会抛异常、不会重试。调了 CAS 却不检查返回值、直接往下走，等于把乐观锁当成必定成功的写操作，冲突时业务悄悄跑偏。标准写法是 `if (!ref.compareAndSet(old, next))` 后重试或走降级分支，返回值是 CAS 语义的一半。

【小技巧】`AtomicIntegerFieldUpdater` 的字段必须是 volatile 且不能是 static，否则 `newUpdater` 直接抛异常；private 字段的 updater 只能在字段所在的类里建，跨类建会因访问权限失败。原文第 2 节只给了建 updater 的例子，这三个建不出来的坑是排查“updater 一建就炸”时先看的点——字段声明本身要先合规。

【小技巧】`AtomicStampedReference` 的 CAS 要先 `getStamp()` 拿到当前邮戳再传进去，写死一个 stamp 值会一直失败：`int stamp = ref.getStamp(); ref.compareAndSet(oldRef, newRef, stamp, stamp + 1)` 才是正确姿势。邮戳是每次成功 CAS 才加一的计数器，调用方自己维护不了，只能每次现取现用。

【小技巧】`LongAdder` 想“读数并清零”用 `sumThenReset()`，别写 `sum()` 后再 `reset()`：两步之间其他线程的累加会丢——sum 读完、reset 之前加进来的数直接被清掉。sumThenReset 把“汇总 + 清零”包成一次操作，统计窗口滚动（比如每分钟出一个 QPS 数）时用它，读到的数才是完整的窗口内计数。

【小技巧】`AtomicReference` 里装可变对象时，别 `get()` 出来改字段再 `set()` 回去：两步之间别的线程可能已经换了引用，你的 set 会把别人的更新覆盖掉。正确做法是装不可变对象、用 `updateAndGet(old -> 新对象)` 一步算出新值，或者自己写 CAS 循环；原子性只保“引用替换”这一步，不保“取出来改完再放回去”这两步。

【小技巧】CAS 自旋循环里别空转，失败重试时加一行 `Thread.onSpinWait()`：Java 9 引入的这个方法告诉 CPU“我正在自旋等别的线程”，CPU 可以省电、让出执行资源，比纯空转更友好。只在确实要忙等的地方用，别把它当 sleep 的替代品——它不释放 CPU 时间片，只是给硬件一个提示；原文第 3 节的退避手段里，它是最轻量的一种。

【小技巧】`AtomicBoolean` 的 `compareAndSet(false, true)` 是最干净的一次性开关：返回 true 的线程就是第一个抢到的，后到的全返回 false，不用再配锁或状态位。初始化只跑一次、定时任务只启动一次、防重复提交的内存版，都可以一个 AtomicBoolean 搞定；抢到之后记得在 finally 或完成时把它复位，否则开关永远关不上。

【冷知识】CAS 在 x86 上就是一条 `LOCK CMPXCHG` 指令：Java 层的 `compareAndSet` 最终落到 CPU 的这条原子指令，LOCK 前缀锁住缓存行、保证多核下“比较 + 替换”不可分割。理解这一点就明白为什么 CAS 不需要操作系统介入、为什么它比 synchronized 轻——它连内核都没进，只是一条带锁前缀的 CPU 指令。

【冷知识】JDK 9 起原子类内部改用 `VarHandle` 实现：`AtomicInteger`、`AtomicLong` 这些类的底层不再直接调 `Unsafe`，而是走 `VarHandle` 的 `compareAndSet`，语义等价、类型更安全。业务代码不用跟着改——`java.util.concurrent.atomic` 的 API 没变，变的是它内部的实现；原文第 1 节说“JDK 9+ 官方替代为 VarHandle”，指的就是这层替换。

【冷知识】`LongAdder` 的 Cell 数组最多扩到 CPU 核数：线程按探测哈希落到不同 Cell，冲突多了才扩容，扩到核数就不再扩——再多 Cell 也没有更多并行收益，只会让 `sum()` 汇总时多跑几个格子。高并发计数场景下，核数就是它分散的上限，机器核数没变，Cell 数也不会无限制涨。

## 7. 面试与自测

- 手写一个基于 CAS 的自增，并说明失败重试的成本从哪来。
- ABA 是什么？发生在哪些数据结构上最危险？两种解法？
- AtomicLong 与 LongAdder 的适用场景如何分界？

### 参考答案

1. 手写自增骨架是自旋重试：do { old = v.get(); } while (!v.compareAndSet(old, old + 1)); 返回 old + 1。失败重试的成本：每次失败都白做一次读旧值、算新值；高竞争下多线程 CAS 同一个变量，缓存行反复失效，大部分尝试都在失败重试，烧 CPU。
2. ABA 是指线程读到值 A，期间别人把 A 改成 B 又改回 A，本线程 CAS 仍会成功——值没变，但“过程”变了。最危险的是把值当作状态历史的场景：栈顶指针、引用复用、无锁链表/栈。解法一是 AtomicStampedReference，给值配版本号，值与版本一起 CAS；解法二是 AtomicMarkableReference，配布尔标记，能判断“变过”但不能计次，适合只关心是否变过的场景；根本上应避免对 ABA 敏感的结构复用。
3. 分界看竞争强度与读精度：AtomicLong 适合低中竞争、需要精确读-改-写的场景；LongAdder 适合高并发只增计数、容忍 sum() 读到近似值的场景，如统计 QPS、埋点计数、批处理进度。LongAdder 把计数分散到多个 Cell 各自 CAS、sum() 再汇总，换的是写入吞吐；需要精确值时不要用它。

## 8. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 悲观路线对照：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- 分段思想应用：[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)
- AQS 里的 CAS：[AQS](开发语言/Java/并发/AQS.md)
