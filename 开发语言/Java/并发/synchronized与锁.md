---
知识点: synchronized与锁
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java-并发编程
  - AQS
  - volatile与JMM
标签:
  - Java
  - 并发
  - 原理
---

# synchronized 与锁

## 1. 核心概念

- 锁解决的是互斥：同一时刻只允许一个线程进入临界区，保护共享可变状态的复合操作（检查再执行、读-改-写）。
- 两条路线：`synchronized` 是 JVM 内置锁（关键字，自动释放，不会漏）；JUC 的 `Lock` 接口以 [AQS](开发语言/Java/并发/AQS.md) 为底座，功能更丰富（可超时、可中断、可公平、多条件队列）。
- 选型口诀：默认 `synchronized`；需要 `tryLock`、公平、多 `Condition` 时才上 `ReentrantLock`。

## 2. synchronized 基础

```java
class Counter {
    private int n = 0;
    public synchronized void add() { n++; }            // 锁 this
    public void addBlock() {
        synchronized (this) { n++; }                   // 语句块：只锁关键几行
    }
    public static synchronized void addStatic() { }     // 锁 Class 对象
    public int get() { return n; }
}
```

- 语义：互斥 + 可见性（解锁 happens-before 后续加同一把锁，见 [volatile与JMM](开发语言/Java/并发/volatile与JMM.md)）+ 可重入（同一线程可重复进同一把锁，内部计数）。
- 锁对象必须固定且共用：`synchronized (new Object())` 每次都是新锁等于没锁；锁字符串字面量、Integer 缓存对象同理危险。
- 底层：对象头 Mark Word 记录锁状态，JDK 6 起有偏向锁 → 轻量级锁（CAS 自旋）→ 重量级锁（阻塞进操作系统互斥量）的升级路径；注意 JDK 15 起偏向锁默认禁用、JDK 18 已废弃，不要再拿它当调优手段。

## 3. ReentrantLock 与 Condition

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();          // 必须放 finally，异常路径漏 unlock = 后来者全死等
}

if (lock.tryLock(200, TimeUnit.MILLISECONDS)) {   // 可超时：防死等，也是死锁的逃生门
    try { /* ... */ } finally { lock.unlock(); }
}
```

比 `synchronized` 多出来的能力：

- `tryLock()` / 带超时：抢不到就走降级逻辑，而不是无限 BLOCKED。
- 可中断加锁 `lockInterruptibly()`：等待时能响应中断取消。
- 公平锁选项：按排队顺序拿锁，防线程饿死，代价是吞吐下降，默认非公平。
- 多个 `Condition`：一个锁配多个等待队列（如队列「非空」「非满」分开唤醒），`synchronized` 只有一个 wait 集合只能 `notifyAll` 乱醒。

**读写锁 `ReentrantReadWriteLock`**：读锁共享、写锁独占，读多写少（本地缓存、配置字典）时读并发度大幅高于一把大锁；注意写锁饥饿与锁降级（持写锁再拿读锁可以，持读锁升写锁不行）。JDK 8 的 `StampedLock` 提供乐观读，适合读极多写极少，但它不可重入、使用更易错，按需选用。

## 4. 死锁：成因与排查

死锁四条件：互斥、持有并等待、不可剥夺、循环等待。破坏任意一条即可，工程上最常用「全局固定加锁顺序」破坏循环等待、「tryLock 带超时」破坏无限等待。

```java
// 经典互锁：转账双方各锁自己的账户再锁对方的
// 解法一：按账户 id 大小统一加锁顺序
// 解法二：tryLock 两把锁，都拿到才执行，拿不齐就释放已拿到的重试
boolean gotA = a.lock.tryLock(100, TimeUnit.MILLISECONDS);
boolean gotB = gotA && b.lock.tryLock(100, TimeUnit.MILLISECONDS);
if (gotA && gotB) {
    try { transfer(a, b, amt); } finally { b.lock.unlock(); a.lock.unlock(); }
} else {
    if (gotA) a.lock.unlock();   // 拿不齐就全部释放，稍后重试
}
```

线上排查流程：

```bash
jstack <pid> | grep -A 10 "Found one Java-level deadlock"   # 直接给出死锁环
# 或用 jconsole / Arthas thread -b 查看 BLOCKED 线程在等哪把锁、被谁持有
```

- 先留现场再重启，别急着重启把证据冲掉；jstack 输出里顺着「waiting to lock ... owned by」就能画出等待环。
- 预防清单：锁粒度尽量小、持锁期间不调外部方法（RPC、回调、发消息）、多把锁按固定顺序、能超时就别无限等。

## 5. 实战应用

- **项目落点**：审批流里「同一笔申请单的状态流转」要防并发重复提交：对单据号维度加锁（或数据库乐观锁 version 字段，二选一；跨实例部署时进程内锁无效，必须用分布式锁或数据库锁，别用本地锁自我安慰）。
- 临界区只放最必要的几行：组装参数、发请求都放锁外；持锁调网络是把临界区放大到网络延迟量级，并发度直接归零。
- Spring 单例 Bean 的字段就是全应用共享状态：往单例里塞可变字段又不加锁，是 Web 项目最常见的串数据事故；要么字段无状态化，要么加锁/换并发容器。

## 6. 常见误区

- 以为静态方法锁与实例方法锁互斥：它们锁的是 Class 与 this 两个不同对象，互不影响。
- `unlock` 不写 finally：临界区抛异常后锁永不释放，后续线程全挂，这类问题在压测下才暴露。
- 锁的范围套住整个方法图省事：含远程调用的方法整体加锁，等于把系统改成单线程。
- 混用两套锁保护同一状态：一处用 `synchronized`、一处用 `ReentrantLock` 保护同一字段——两套机制互不知晓，等于没锁。

## 小技巧与冷知识

【小技巧】锁对象用私有的 final Object，别直接锁 this：`synchronized (this)` 的锁是公开的，任何拿到你对象引用的外部代码也能 `synchronized (你的对象)`，锁竞争和死锁的源头就藏在类外面。写成 `private final Object lock = new Object()` 再 `synchronized (lock)`，锁的可见范围就收进了类内部，别人的代码想插手也插不进来；原文第 2 节只说了锁对象要固定共用，这条是“共用给谁看”的下一层。

【小技巧】`wait()` 必须写在 while 循环里，不能写 if：线程可能在条件还没满足时被“虚假唤醒”（spurious wakeup），醒来就直接往下走会读到不满足条件的状态。标准写法是 `while (!condition) { obj.wait(); }`，醒来先复查条件，不满足就继续等；只用 if 判断一次等于默认“每次醒来条件都成立”，这是 wait/notify 代码里最常见的坑。

【小技巧】一个 monitor 上挂着多个等待条件时，用 `notifyAll()` 别用 `notify()`：`notify()` 只唤醒一个任意线程，如果唤醒的是在等另一个条件的线程，它醒来发现条件不对又睡回去，这次信号就丢了，等真正该醒的线程一直没人叫。只有一个等待条件时 `notify()` 才安全；条件一多就用 `notifyAll()`，让所有等待线程自己醒来复查条件，原文第 3 节说 synchronized 只有一个 wait 集合只能 notifyAll 乱醒——这里是“乱醒”反而比“叫错人”安全。

【小技巧】`wait()` 会把锁的重入次数一次性全放掉：一个线程重入同一把锁 3 次后调用 `wait()`，锁是彻底释放、不是只减一次，别的线程能完整拿到锁；它被唤醒后要按原重入次数把锁重新拿满才返回。排障时看到“重入的线程 wait 后别人进来了”别当成锁坏了，这是 wait 的语义——条件等待和锁是“整借整还”，不是按次借还。

【小技巧】子类重写 synchronized 方法不会自动继承锁：`synchronized` 不是方法签名的一部分，父类方法标了 synchronized，子类重写时没写这个关键字，覆盖后的方法就是不加锁的。改父类实现时如果在子类里重写了同步方法，记得把 synchronized 补上；只看父类签名以为“这个方法天生是同步的”，重构后并发问题就悄悄回来了。

【小技巧】StampedLock 的乐观读必须配 `validate()` 校验：`tryOptimisticRead()` 拿到的 stamp 只是“读到的数据可能有效”，读完要用 `validate(stamp)` 确认期间没有写锁进来过，校验失败就升级成 `readLock()` 重读。只拿 stamp、不校验，等于读了一段随时可能被改掉的数据还没发现；原文第 3 节只说了 StampedLock 提供乐观读，这行校验才是乐观读成立的前提。

【小技巧】程序里也能主动查死锁，不用等线上 jstack：`ManagementFactory.getThreadMXBean().findDeadlockedThreads()` 会返回陷入死锁的线程 id 数组，没死锁时返回 null。健康检查或定时监控里定期调一次，死锁刚发生就能告警并定位到线程；等用户报障再去 jstack 留现场，中间已经丢了一段时间。

【小技巧】`ReentrantLock.getHoldCount()` 能看当前线程把同一把锁重入了几次：怀疑“unlock 少写了一次、锁一直没放”时，在关键位置打一行这个值，返回几就说明当前线程还持有几重。配合 `isHeldByCurrentThread()` 一起用，比猜“锁是不是我这边没放”可靠；它只统计当前线程的持有次数，别的线程的持有情况它看不到。

【冷知识】`synchronized` 编译后是两条字节码指令：方法体或代码块里的同步，javap -c 能看到 `monitorenter` 和 `monitorexit`。编译器还会给临界区加一张异常表，保证方法里抛异常时也走 `monitorexit` 把锁放掉——这就是 synchronized“自动释放、不会漏”的底层来源，不是 JVM 运行时额外做的善后。

【冷知识】monitor 这个概念来自 Hoare 1974 年的论文《Monitors: an operating system structuring concept》：Java 是最早把 monitor 做进语言关键字的主流语言之一，`synchronized` + `wait/notify` 就是论文里 monitor 的直接落地。理解这一点就明白为什么 wait/notify 必须在 synchronized 块里调——monitor 的语义本来就要求“先持有监视器，才能等它的条件”。

【冷知识】两把锁没有天然顺序时，按对象身份排加锁顺序是《Java 并发编程实战》给的经典解法：比较 `System.identityHashCode(a)` 和 `System.identityHashCode(b)`，小的先锁、大的后锁；两个哈希值撞了（概率极低），再加一把全局的 tie 锁兜底。原文第 4 节只给了“按账户 id 大小统一顺序”，这条是“连 id 都没有、只有两个对象时”的通用版。

## 7. 面试与自测

- `synchronized` 与 `ReentrantLock` 的区别，从实现层（JVM 内置 vs AQS）说起。
- 公平锁为什么吞吐低？读写锁为什么不能由读锁升级写锁？
- 讲一次你排查死锁的步骤：拿什么、看什么、怎么定位到代码行。

### 参考答案

1. synchronized 是 JVM 内置锁：关键字、自动释放，底层是对象头 Mark Word 的锁状态升级（偏向→轻量→重量；JDK 15 起偏向默认禁用、JDK 18 已废弃）。ReentrantLock 是 JUC 的 Lock，以 AQS 为底座：state + FIFO 队列管排队、阻塞、唤醒。能力上后者多出 tryLock/带超时、可中断加锁、公平选项、多个 Condition；选型口诀是默认 synchronized，需要这些能力时才上 ReentrantLock。
2. 公平锁吞吐低：每次唤醒都严格按排队顺序，线程切换更频繁，非公平允许后来者直接抢、减少切换。读锁不能升级写锁：拿写锁要求当前没有读锁持有者，而升级线程自己就持有一把读锁，永远等不到自己释放，会死锁；反过来持写锁再拿读锁（锁降级）是允许的。
3. 排查步骤：先留现场再重启。jstack <pid>，看“Found one Java-level deadlock”直接给出的死锁环，或用 jconsole / Arthas thread -b 看 BLOCKED 线程在等哪把锁、被谁持有；顺着“waiting to lock … owned by”画出等待环，定位到代码行。预防：全局固定加锁顺序、tryLock 带超时、锁粒度尽量小、持锁期间不调外部方法。

## 8. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 锁的底座：[AQS](开发语言/Java/并发/AQS.md)
- 内存语义：[volatile与JMM](开发语言/Java/并发/volatile与JMM.md)
- 无锁路线：[CAS与原子类](开发语言/Java/并发/CAS与原子类.md)
