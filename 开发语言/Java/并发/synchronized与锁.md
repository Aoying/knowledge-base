---
知识点: synchronized与锁
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-23
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

## 7. 面试与自测

- `synchronized` 与 `ReentrantLock` 的区别，从实现层（JVM 内置 vs AQS）说起。
- 公平锁为什么吞吐低？读写锁为什么不能由读锁升级写锁？
- 讲一次你排查死锁的步骤：拿什么、看什么、怎么定位到代码行。

## 8. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 锁的底座：[AQS](开发语言/Java/并发/AQS.md)
- 内存语义：[volatile与JMM](开发语言/Java/并发/volatile与JMM.md)
- 无锁路线：[CAS与原子类](开发语言/Java/并发/CAS与原子类.md)
