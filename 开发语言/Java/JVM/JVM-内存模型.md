---
知识点: JVM 内存模型
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java-并发编程
  - JVM-GC调优
  - Java
标签:
  - Java
  - JVM
  - 并发
---

# JVM 内存模型

## 1. 核心概念

先分清两个概念，别混：

| 概念 | 英文 | 管什么 | 所属规范 |
| --- | --- | --- | --- |
| Java 内存模型 | JMM, Java Memory Model | 多线程可见性、有序性、原子性 | JSR-133，JDK 5 起生效 |
| JVM 运行时内存区域 | Runtime Data Areas | 堆、虚拟机栈、方法区、程序计数器、本地方法栈的划分与溢出 | JVM 规范，GC 篇讲它 |

本文只讲前者 JMM。堆/栈/方法区去看 JVM-GC调优。

为什么需要 JMM：

- CPU 有 L1/L2/L3 缓存，线程读到的可能是缓存旧值，不是主内存新值。
- CPU 有写缓冲区，写操作先入缓冲区，延迟刷回，其他核看不到。
- 编译器重排序：不改变单线程语义时，编译器可调整指令顺序。
- 处理器重排序：乱序执行、流水线导致指令完成顺序与书写顺序不一致。
- 结果：一个线程的写，另一个线程不一定看得见；看得见，也不一定按顺序看得见。

JMM 的作用：屏蔽硬件差异，给出统一规则。遵守规则，程序结果可预期；不遵守，数据竞争，结果不可预期。

## 2. 基础知识

### 2.1 主内存与工作内存抽象

- 主内存：对应物理主存抽象，所有线程共享，变量主副本在此。
- 工作内存：对应 CPU 缓存 + 寄存器 + 写缓冲区抽象，每线程私有。
- 线程读写变量，必须经工作内存：先从主内存加载副本，修改后刷回主内存。
- 线程间传值，必须经主内存中转，不能直连对方工作内存。

该抽象是逻辑模型，不对应某个硬件部件，别拿它和堆/栈对应。

### 2.2 三大特性与保证手段

| 特性 | 含义 | 保证手段 |
| --- | --- | --- |
| 原子性 | 操作不可分割，要么全做，要么不做 | `synchronized`、`Lock`、原子类 `Atomic*`；long/double 非原子读写需 `volatile` 或加锁（64 位 JVM 多已原子，仍按规范处理） |
| 可见性 | 一线程修改，其他线程能及时看到 | `volatile` 写刷主内存、读读主内存；`synchronized` 解锁前刷、加锁后拉；`final` 正确构造后可见 |
| 有序性 | 本线程内看有序，跨线程看可能乱序，需约束重排 | `volatile` 禁重排、`synchronized` 同一时刻单线程进入、`happens-before` 规则 |

注意：`i++` 不是原子操作，分读、加、写三步。任何特性都不能只靠普通读写保证。

## 3. 进阶知识

### 3.1 volatile 语义

- 可见性：写立即刷主内存，读直接读主内存，不用缓存旧值。
- 有序性：禁止相关重排序，靠插入内存屏障实现，见第 4 章。
- 不保证原子性：复合操作仍会竞态。

反例，计数必错：

```java
public class VolatileCounter {
    public volatile int count = 0;
    public void inc() { count++; } // 读-改-写三步，非原子
    public static void main(String[] args) throws Exception {
        VolatileCounter c = new VolatileCounter();
        Thread[] ts = new Thread[10];
        for (int i = 0; i < 10; i++) {
            ts[i] = new Thread(() -> { for (int j = 0; j < 1000; j++) c.inc(); });
            ts[i].start();
        }
        for (Thread t : ts) t.join();
        System.out.println(c.count); // 期望 10000，实际常小于 10000
    }
}
```

适用场景：一写多读、状态标志位、触发器。不适用：计数、余额、依赖旧值的更新，这类用锁或原子类。

正确停止标志写法见 5.2。

### 3.2 final 域语义

- 构造内对 `final` 域的写，happens-before 构造结束后对象引用被其他线程看到。
- 前提：构造过程中对象引用不能逸出（不能在构造里把 `this` 注册到静态集合、启动线程、传给外部）。
- 满足前提，其他线程看到对象引用时，必能看到已初始化的 `final` 字段，无需同步。
- `final` 引用指向的对象，其字段在构造后冻结的只是引用本身；对象内部可变字段后续修改不自动可见，仍需同步。

```java
public final class Config {
    public final int timeout;
    public Config() { this.timeout = 3000; } // 构造完成后对其他线程可见
}
```

## 4. 原理剖析

### 4.1 happens-before 八大规则

happens-before 是 JMM 的核心契约：若 A happens-before B，则 A 的结果对 B 可见，且 A 的执行在内存语义上先于 B。它不等于时间先后：时间在前不一定 happens-before，happens-before 也不要求物理时间紧挨。

| # | 规则 | 说明 |
| --- | --- | --- |
| 1 | 程序顺序规则 | 同一线程内，前面的操作 happens-before 后面的操作 |
| 2 | 监视器锁规则 | 对锁的 unlock happens-before 后续对同一锁的 lock |
| 3 | volatile 规则 | volatile 写 happens-before 后续对同一变量的 volatile 读 |
| 4 | 线程启动规则 | `Thread.start()` happens-before 线程内所有操作 |
| 5 | 线程终止规则 | 线程内所有操作 happens-before 其他线程 `join()` 成功返回，或检测到线程终止 |
| 6 | 线程中断规则 | `interrupt()` happens-before 线程检测到中断（`isInterrupted`/`InterruptedException`） |
| 7 | 对象终结规则 | 构造完成 happens-before `finalize()` 开始 |
| 8 | 传递性 | A hb B 且 B hb C，则 A hb C |

重点理解：

- 没有 happens-before 关系约束的两个操作，就是数据竞争，顺序和可见性都不保证。
- 程序顺序规则只保单线程视角：编译器和 CPU 仍可重排，只要单线程语义不变（as-if-serial）。
- 锁规则和 volatile 规则是跨线程通信的桥：写方释放，读方获取，中间靠传递性把写前所有普通写都带过去可见。

### 4.2 内存屏障

屏障是 CPU 指令，禁止跨屏障重排，并强制刷/失效缓存。

| 屏障 | 约束 | 作用 |
| --- | --- | --- |
| LoadLoad | Load1; LoadLoad; Load2 | 保证 Load1 先于 Load2 读 |
| StoreStore | Store1; StoreStore; Store2 | 保证 Store1 先刷出，再执行 Store2 |
| LoadStore | Load1; LoadStore; Store2 | 保证 Load1 先于 Store2 刷出 |
| StoreLoad | Store1; StoreLoad; Load2 | 保证 Store1 对所有线程可见后，再执行 Load2，最重、开销最大 |

volatile 插入策略（JSR-133 模型，简化）：

| 位置 | 插入屏障 | 目的 |
| --- | --- | --- |
| volatile 写之前 | StoreStore | 前面普通写先刷出，不与 volatile 写重排 |
| volatile 写之后 | StoreLoad | volatile 写全局可见后，后面读才执行 |
| volatile 读之前 | LoadLoad + LoadStore | 后面读写不提前到读之前 |
| volatile 读之后 | LoadLoad + LoadStore | 同上，防后续操作重排到读前 |

x86 上很多屏障是 no-op，只有 StoreLoad 需真实指令（如 `lock` 前缀），所以 volatile 读便宜、写较贵。其他架构成本不同，写代码不要依赖硬件细节。

## 5. 实战应用

### 5.1 双重检查锁定单例必须加 volatile

对象创建三步：1 分配内存，2 调用构造初始化，3 引用指向内存。第 2、3 步可重排为 1、3、2。

无 `volatile` 时，线程 A 执行到第 3 步未初始化完，线程 B 在第一次检查看到引用非 null，直接返回半初始化对象，读字段得默认值或出错。

```java
public class Singleton {
    private static volatile Singleton instance; // 必须 volatile
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {                 // 第一次检查，无锁快速路径
            synchronized (Singleton.class) {
                if (instance == null) {         // 第二次检查，持锁防重复创建
                    instance = new Singleton(); // volatile 写禁止 2、3 重排
                }
            }
        }
        return instance;
    }
}
```

JDK 5 前 volatile 语义弱，DCL 失效；JDK 5 后 JSR-133 修复，才可用。更简单替代：静态内部类懒加载或枚举单例，无需 DCL 时优先用它们。

### 5.2 状态标志位

```java
public class Worker {
    private volatile boolean running = true;
    public void stop() { running = false; }
    public void run() {
        while (running) { /* 干活 */ } // volatile 保证能读到 stop 的修改
    }
}
```

一写多读场景典型用法：开关、配置版本号、取消令牌。

### 5.3 synchronized 与框架中的 volatile

- `synchronized` 可见性来源：unlock happens-before 后续 lock，解锁前把工作内存刷回， 加锁后清空工作内存重读。所以锁内改的普通变量，下一持锁线程可见。
- AQS：`state` 是 `volatile int`，靠 CAS 改它同步队列状态，屏障语义由 volatile + CAS 提供。
- `ConcurrentHashMap`：Node 的 `val`、`next` 用 `volatile`，保证扩容、并发读写时链表/红黑树节点更新可见，细节看并发编程篇。
- 原则：先想清 happens-before 链再写代码，不要靠 `sleep` 或延时碰运气修复可见性。

## 6. 疑难杂症

### 6.1 停止标志不退出：JIT 提升

现象：普通 `boolean` 标志，主线程改了，工作线程 `while` 循环永不退出。`-client` 模式偶尔正常，`-server` 模式必现。

原因：无 volatile 时无可见性保证；JIT 发现循环内不改该变量，将其提升到寄存器/改写为 `while(true)` 等价形式，后续主内存更新永远读不到。

排查：打印线程堆栈确认卡在循环；改 `volatile` 或用 `AtomicBoolean` 后消失即实锤。别用加 `System.out.println` 掩盖——打印自带同步会偶然修复，造成假象。

修复：标志位加 `volatile`，或循环内加锁读取。

### 6.2 读到默认值：不安全发布

现象：A 线程创建对象并发布到静态变量，B 线程读到对象非 null，但 int 字段是 0、引用字段是 null，与构造赋值不符。

```java
class Holder { int x; Holder() { x = 42; } }
static Holder holder; // 非 volatile，无锁发布，不安全
// 线程A: holder = new Holder();
// 线程B: if (holder != null) System.out.println(holder.x); // 可能打印 0
```

原因：引用赋值与字段初始化重排，或字段写未刷主内存，且双方无 happens-before。

修复三选一：`volatile` 发布、加锁发布、用 `final` 字段且构造不逸出。集合类发布同理，优先用 `ConcurrentHashMap` 等线程安全容器。

### 6.3 排查思路汇总

1. 先问有没有 happens-before 链，没有就是代码 bug，不是玄学。
2. 复现要开多线程、高迭代、server 模式，单次运行正常不证明正确。
3. 用 `jstack` 看线程状态，用压力测试（如 jcstress）验证并发语义，别只靠肉眼。
4. 区分可见性问题与原子性问题：看不到是前者，计数丢更新是后者，药不同。

## 7. 常见误区

| 误区 | 正解 |
| --- | --- |
| happens-before 就是时间先后 | 它是可见性契约，逻辑关系，与时钟无关 |
| volatile 能保证计数正确 | 只保可见与有序，不保复合操作原子性，计数用 `AtomicInteger` 或锁 |
| 单线程不用管重排序 | as-if-serial 只保单线程结果等价，跨线程无此保证 |
| 加了 synchronized 就不用 volatile | 锁只在临界区进出时同步，锁外读普通变量仍可能读旧值 |
| volatile 变量读写和锁一样贵/一样轻 | 读近普通读，写含 StoreLoad 较贵；且 volatile 不阻塞、不替代互斥 |

## 8. 性能优化

- 能用 volatile 别用锁：无竞争、无阻塞，适合标志位；需互斥或复合操作时别硬套。
- 缩小同步块，只包共享变量操作，IO 和耗时计算放锁外。
- 读写分离：读多写少考虑读写锁或不可变对象 + volatile 引用整体替换。
- 避免伪共享：高频写的不同变量落在同一缓存行会互相失效，JDK 8 用字段填充、JDK 9+ 用 `@Contended`（需 JVM 参数）。
- 不要滥加 volatile：每个 volatile 写都有屏障成本，高频写字段评估用原子类分段或 LongAdder。

## 9. 版本差异

| 版本 | 变化 |
| --- | --- |
| JDK 5 | JSR-133 落地，volatile/final 语义增强，DCL 才成立 |
| JDK 6-7 | JIT 对无同步代码的激进优化增多，老代码可见性 bug 更易暴露 |
| JDK 8 | 无 JMM 模型变更，`LongAdder`、`CompletableFuture` 等基于既有语义构建 |
| JDK 9+ | VarHandle 提供 acquire/release/opaque 四种内存语义，比 volatile 更细粒度 |
| JDK 17/21 | 模型本身不变，虚拟线程不改变可见性规则，共享变量仍需同步 |

## 小技巧与冷知识

【小技巧】`volatile` 修饰数组只保护引用，不保护元素：`volatile int[] arr` 换一个新数组引用时可见，但 `arr[0] = 1` 这种元素写不保证对其他线程可见。元素也要跨线程可见时，用 `AtomicIntegerArray`，或者把数组整体换掉、只用 volatile 引用发布新数组；数组和它的元素是两层东西，volatile 只管最外面那一层。

【小技巧】别把 volatile 标志先缓存到局部变量再进循环：`boolean r = running; while (r) { ... }` 里，r 只在循环前读了一次，之后主线程把 running 改成 false，循环照样不退出。条件里每次直接读 `while (running)`，让每次判断都走一次主内存；原文 6.1 的 JIT 提升是编译器干的，这条是自己手写出来的同一个坑。

【小技巧】一次性启动、一次性关闭用 `AtomicBoolean.compareAndSet`：`if (!started) { started = true; 启动(); }` 里两个线程可能同时通过检查，各启动一次。`started.compareAndSet(false, true)` 是原子地"检查并翻转"，只放一个线程进去，返回值直接告诉你是不是第一个；volatile boolean 只能保证"看到最新值"，保证不了"只有一个线程通过检查"。

【小技巧】发布对象时，把 volatile 标志写在最后一行：先写完所有普通字段，最后再写 `ready = true`（volatile）。读方读到 ready 是 true 时，靠 happens-before 传递性，ready 写之前的所有普通字段写也一起可见。顺序反过来——先写标志、再写字段——读方可能看到标志已就绪、字段还是默认值；发布的"最后一笔"必须是 volatile 写。

【小技巧】自旋等一个标志时，循环里加一行 `Thread.onSpinWait()`：它告诉 CPU"我在等别的线程改个值"，CPU 可以省电、让出执行资源，比纯空转更友好。只在确实要忙等的地方用，别把它当 sleep 的替代品——它不释放 CPU 时间片，只是给硬件一个提示；Java 9 才引入，Java 8 的项目照抄编译不过。

【小技巧】锁里读普通变量就够了，不用再套一层 volatile：`synchronized` 进出本身就同步可见性，锁内读到的普通变量已经是最新的，再给字段加 volatile 只是多一层屏障成本。volatile 留给锁外的场景——标志位、发布、安全发布；锁内用普通变量，锁外才轮到 volatile 上场，分工别混。

【冷知识】happens-before 这个概念不是 Java 发明的：它来自 Leslie Lamport 1978 年的论文《Time, Clocks, and the Ordering of Events》，原意是给分布式系统里的事件定一个"谁先发生"的偏序关系。Java 内存模型把它借了过来，当作可见性的契约——"A happens-before B"在论文里是逻辑时钟的排序依据，在 JMM 里变成了"读到 A 的结果就能看到 B 之前所有写"的保证。

【冷知识】JSR-133 由马里兰大学的 Bill Pugh 牵头，2004 年随 JDK 5 落地：修的正是 JDK 5 之前 volatile 语义太弱的老问题——那时 volatile 只保证"写了马上能看到"，不禁止重排序，双重检查锁定写了 volatile 照样可能读到半初始化对象。JSR-133 把"禁止相关重排序"写进 volatile 语义，DCL 才从"公认的反模式"变回可用写法；原文 5.1 说"JDK 5 前 DCL 失效"，根子就在这次修订。

【小技巧】`static final` 字段是最省事的安全发布：类初始化由 JVM 加锁保证只跑一次，`static final Config CFG = new Config(...)` 写好后，所有线程读到的都是初始化完成的值，不用 volatile、不用加锁。配置对象、单例、常量表这类"初始化一次、之后只读"的东西，能放 static final 就别自己搭发布逻辑；JVM 的类加载锁已经替你把 happens-before 链搭好了。

## 10. 面试与自测

1. volatile 能保证原子性吗？不能，举 `i++` 说明读-改-写三步竞态。
2. DCL 为什么加 volatile？防对象创建第 2、3 步重排导致半初始化对象被读。
3. happens-before 八规则里，哪两条是跨线程同步的核心？锁规则、volatile 规则。
4. synchronized 的可见性怎么来的？unlock hb 后续 lock，靠传递性带出锁内所有写。
5. final 字段为什么线程安全？构造内写 hb 引用逸出，前提是构造不逸出 `this`。
6. 线程 A 写普通变量后写 volatile 变量，线程 B 读到 volatile 新值，能否看到普通变量新值？能，靠 StoreStore 屏障 + 传递性。自答不出第 6 题，回看 4.1、4.2。

### 参考答案

1. 不能。volatile 只保证可见性与有序性，不保证原子性；`i++` 是读、改、写三步，两个线程交错时会互相覆盖更新。正文反例：10 个线程各加 1000 次，期望 10000，实际常小于 10000。计数、余额这类依赖旧值的更新要用锁或 `AtomicInteger`。

2. 对象创建分三步：分配内存、调用构造初始化、引用指向内存，其中第 2、3 步可被重排为 1、3、2。无 volatile 时，线程 A 执行到第 3 步、对象尚未初始化完，线程 B 第一次检查看到引用非 null 就会直接返回半初始化对象。`instance` 加 volatile 后，volatile 写禁止 2、3 重排；注意这依赖 JDK 5 起 JSR-133 增强后的语义，JDK 5 前 DCL 失效。

3. 监视器锁规则与 volatile 规则。锁规则：对同一锁的 unlock happens-before 后续 lock；volatile 规则：volatile 写 happens-before 后续对同一变量的 volatile 读。两者是跨线程通信的桥，写方释放、读方获取，再靠传递性把写前所有普通写带过去可见。

4. 解锁前把工作内存刷回主内存，加锁后清空工作内存重新读取；且 unlock happens-before 后续对同一锁的 lock。靠传递性，锁内修改的普通变量对下一个持锁线程可见。但注意：锁只在临界区进出时同步，锁外读普通变量仍可能读旧值。

5. 构造内对 final 域的写 happens-before 构造结束后对象引用被其他线程看到，因此其他线程看到引用时必能看到已初始化的 final 字段，无需同步。前提是构造过程中对象引用不能逸出，即不能在构造里把 `this` 注册到静态集合、启动线程或传给外部。另注意：final 冻结的只是引用本身，其指向对象的可变字段后续修改不自动可见，仍需同步。

6. 能。线程 A 写普通变量在前、写 volatile 变量在后，volatile 写之前插入 StoreStore 屏障，保证前面普通写先刷出；volatile 写 happens-before 线程 B 对同一变量的 volatile 读。再靠传递性，线程 B 读到 volatile 新值时，也能看到该 volatile 写之前的所有普通写新值。

## 11. 关联知识

- Java-并发编程：锁、AQS、线程池、并发容器的实现都建立在 JMM 之上。
- JVM-GC调优：讲运行时内存区域与 GC，与本文 JMM 是两个维度，面试常连问，注意区分。
- Java：语言层面的 `synchronized`、`volatile`、`final` 关键字语法在此处落到内存语义。
