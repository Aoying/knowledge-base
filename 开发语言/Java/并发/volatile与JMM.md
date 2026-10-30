---
知识点: volatile与JMM
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java-并发编程
  - JVM-内存模型
  - synchronized与锁
标签:
  - Java
  - 并发
  - 原理
---

# volatile 与 JMM

## 1. 核心概念

- 并发三大问题：**原子性**（操作不可分割）、**可见性**（一个线程的修改其他线程何时能看到）、**有序性**（指令实际执行顺序可能与书写顺序不同）。锁一次解决三个，`volatile` 只解决可见性与有序性，不解决原子性。
- JMM（Java 内存模型）是规范层抽象：主内存存共享变量，线程在自己的工作内存里操作副本，JMM 规定了副本与主内存同步的时机与顺序语义，让同一份代码在不同 CPU 架构上行为一致。
- 补充概念见 [JVM-内存模型](开发语言/Java/JVM/JVM-内存模型.md)（运行时数据区与对象内存布局，不要与 JMM 混淆：一个讲内存长什么样，一个讲多线程下读写如何互相看见）。

## 2. happens-before：判断可见性的唯一标尺

不要背内存屏障细节，先会用 happens-before 规则推理：若 A happens-before B，则 A 的结果对 B 一定可见。核心规则：

| 规则 | 含义 |
|---|---|
| 程序顺序 | 同一线程内，书写在前的操作 happens-before 在后的（仅限该线程视角） |
| 锁规则 | 解锁 happens-before 随后对同一把锁的加锁 |
| volatile 规则 | volatile 写 happens-before 随后对同一变量的读 |
| 线程启动 | `start()` 之前的操作 happens-before 新线程内的操作 |
| 线程终止 | 线程内操作 happens-before 其他线程 `join()` 返回后 |
| 传递性 | A hb B 且 B hb C，则 A hb C |

实用的推论：**volatile 的「顺带刷新」**——写 volatile 变量时，它之前所有普通变量的修改会一并刷给后续读到该 volatile 变量的线程。所以「先写数据、再写 volatile 标志位」是经典的发布模式（如配置刷新、初始化完成标记）。

## 3. volatile 的两个作用

```java
class Worker {
    private volatile boolean running = true;   // 停止开关必须 volatile
    void run() {
        while (running) { /* 干活 */ }          // 没有 volatile 时，JIT 可能把读优化成
    }                                           // 只读一次寄存器，永远看不到新值
    void stop() { running = false; }
}
```

1. **保证可见性**：写立即刷新到主内存，读强制从主内存取，不走线程本地缓存。
2. **禁止重排序**：通过插入内存屏障，禁止特定重排。重排来自编译器、JIT 与 CPU——单线程语义不变的前提下它们可以任意调整独立指令的顺序，多线程下没有同步手段时，别人看到的顺序可能就是乱的。

```java
// 有序性的经典翻车：没有 volatile 时，线程 B 可能看到 ready=true 却读到 data=0
int data = 0;
volatile boolean ready = false;
// 线程 A: data = 42; ready = true;      （volatile 写禁止前面的写被排到它后面）
// 线程 B: if (ready) use(data);         // 能读到 42，靠 hb 传递性 + 顺带刷新
```

**边界务必记牢**：`volatile int count; count++` 依然会丢更新——`count++` 是读、加、写三步，volatile 只保证每一步读到最新值，不保证三步不可分割。计数场景用 [CAS与原子类](开发语言/Java/并发/CAS与原子类.md) 或加锁。

## 4. 双重检查锁定（DCL）与单例

```java
public class AppConfig {
    private static volatile AppConfig instance;   // volatile 不可省

    public static AppConfig getInstance() {
        if (instance == null) {
            synchronized (AppConfig.class) {
                if (instance == null) {
                    instance = new AppConfig();
                }
            }
        }
        return instance;
    }
}
```

- 原理：`new` 分三步——分配内存、调用构造初始化、引用指向内存。没有 volatile 时后两步可重排：另一个线程在第一次判空时可能拿到「引用非 null 但对象未初始化完」的半成品。
- volatile 写屏障禁止这种重排，把 DCL 从「看起来对」修成「真的对」。这也是 [单例](开发语言/Java/设计模式/单例.md) 篇里 DCL 写法要求字段必须 volatile 的原因，两边讲的是同一个坑。
- 现代替代：静态内部类 Holder 写法靠类加载的加锁初始化保证安全与懒加载，不想写 DCL 时优先它。

## 5. final 的内存语义

- 正确构造的对象，其 `final` 字段对其他线程可见时保证已初始化完成（构造器里不能让 `this` 逸出，否则该保证作废）。
- 这是不可变对象天然线程安全的根源：状态全 final + 不泄漏 this，发布后任意线程读都是完整一致的快照。设计并发数据结构时，优先不可变 + `AtomicReference` 整体替换，而不是给可变对象层层加锁。

## 6. 常见误区

- 把 volatile 当「轻量级锁」：它不互斥，临界区该加锁还得加锁。
- 用普通 boolean 做跨线程开关（停止标志、配置生效标志）：必翻车，只是时间问题。
- 以为加了 volatile 数组元素也可见：volatile 修饰的是数组引用本身，元素读写不带 volatile 语义，需用 `AtomicIntegerArray` 或对单个元素单独处理。
- 靠 sleep 解决可见性：睡一觉「应该刷新了吧」是时序赌博，与语义保证无关。

## 小技巧与冷知识

【小技巧】停止标志用 volatile 只是第一步，循环体里别空转：`while (running) {}` 这种忙等会把一个核烧满，正确做法是在循环里加 `Thread.sleep()` 或 `LockSupport.parkNanos()`，让出 CPU 再去查标志。volatile 解决的是“能不能看到新值”，不是“等的时候省不省资源”，停表标志和等待策略要分开想。

【小技巧】volatile 的“顺带刷新”只惠及读到那个 volatile 变量的线程：线程 A 写完数据再写 volatile 标志，只有随后读到该标志的线程 B 才能保证看到 A 之前写的数据。另一个线程 C 根本没读这个标志、直接去读数据，它看到的仍可能是旧值；发布通道和读取通道必须配对，单方面写标志不等于向所有线程广播。

【小技巧】配置热刷新别逐字段加 volatile，用 volatile 引用整体替换：把一整组配置装进一个不可变对象，用 `volatile Config current` 持有，刷新时构造新对象、一次赋值替换引用。读方拿到的是某一版完整快照，不会出现“新超时配了旧重试次数”这种半新半旧的组合；逐字段 volatile 每个字段各管各的，拼不出一致性。

【小技巧】volatile 引用只保证引用本身可见，不保证对象内部字段：`volatile User user` 保证别的线程能看到最新的 user 引用，但 user 对象里的普通字段仍可能读到旧值。安全发布的完整链条是“volatile 引用 + 对象内部字段 final 或发布前已写完”，只给引用加 volatile 就去读写可变对象的字段，等于只锁了门没锁窗。

【小技巧】“先查后改”不能用 volatile 代替锁：`if (!flag) { flag = true; doWork(); }` 里查和改是两步，两个线程可能同时查到 false、同时进临界区，volatile 只保证每一步看到最新值，不保证两步之间没人插队。查后改、读后写这类复合操作要上锁或用 CAS 循环，volatile 管的是单次读写的可见性，不是多步组合的原子性。

【小技巧】排查“改了配置没生效”时，先看两处再怀疑框架：一是字段本身是不是 volatile（普通字段的修改可能永远留在写线程的工作内存里），二是读线程有没有把旧值缓存在局部变量里（`Config c = current;` 之后一直用 c，再怎么刷新引用也看不到）。前者是可见性问题，后者是代码把快照钉死了，修的方向完全不同。

【小技巧】happens-before 是可见性保证，不是执行顺序保证：A happens-before B 只说“B 一定能看到 A 的结果”，不说“A 一定比 B 先执行”。推理并发 bug 时别把 hb 当成时序表去排执行先后，它回答的是“谁能看到谁的写”，执行顺序仍由调度决定；拿 hb 去论证“这行一定先跑”会得出错误结论。

【小技巧】构造器里别让 `this` 逸出：构造器中调 `register(this)`、`new Thread(this).start()` 或把 this 传给别的线程，final 字段“构造完才可见”的保证立刻作废，别的线程可能读到半初始化的对象。发布对象要等构造器完全返回后再做，监听器注册、线程启动都挪到构造器外面；原文第 5 节的“不能让 this 逸出”就是这条，逸出的具体姿势就这三种最常见。

【冷知识】现在的 JMM 是 JSR-133 的成果，随 Java 5 在 2004 年发布：旧 JMM 里 volatile 语义有漏洞，DCL 在旧规范下无论如何写都不正确，JSR-133 由 Bill Pugh 等人主导重写，才把 volatile 的 happens-before 规则、final 字段语义和双重检查锁定一次修通。读到 2004 年以前的并发资料要小心，里面的 volatile 结论可能按旧规范写，早已不适用。

【冷知识】`long` 和 `double` 的读写在 32 位 JVM 上可能被拆成两步：JLS 17.7 明确允许非 volatile 的 long/double 读写分两次 32 位操作，一个线程可能读到“高 32 位是新值、低 32 位是旧值”的撕裂值。64 位 JVM 上读写本身就是原子的，但跨平台的代码仍建议把共享的 long/double 标 volatile 或用 AtomicLong——规范留的口子在 32 位环境下是真会踩到的。

【冷知识】volatile 在 C/C++ 里和 Java 里是两个东西：C/C++ 的 volatile 只告诉编译器“别优化掉这次读写”，不提供任何多线程可见性或有序性保证，C++ 靠 `std::atomic` 才解决并发问题。Java 程序员转 C++ 时把“加 volatile 就线程安全”的直觉带过去会直接翻车，反过来 C++ 程序员看 Java 代码也常低估 volatile 的作用——同名不同义。

【冷知识】volatile 的保证不是靠“每次读写都绕过缓存”实现的：底层是内存屏障约束重排序、配合缓存一致性协议让修改对其他核可见，而不是每次读写都去主内存搬数据。把 volatile 理解成“禁止用缓存”会误判性能——它的开销来自屏障禁止的优化和一致性流量，不是“每次都走内存”，读 volatile 在现代 CPU 上仍然可以命中缓存。

## 7. 面试与自测

- happens-before 八条规则说出常用的五条，并用它解释 DCL 为什么需要 volatile。
- volatile 与 synchronized 在可见性实现上的差别是什么？
- 为什么不可变对象不需要同步也能安全发布？

### 参考答案

1. 常用的五条：程序顺序（同线程内书写在前 happens-before 在后）、锁规则（解锁 happens-before 随后对同一把锁的加锁）、volatile 规则（volatile 写 happens-before 随后对同一变量的读）、线程启动（start() 之前的操作 happens-before 新线程内的操作）、线程终止（线程内操作 happens-before 其他线程 join() 返回后），再加传递性串联。DCL 需要 volatile 的原因：new 分三步（分配内存、构造初始化、引用指向内存），无 volatile 时后两步可重排，另一线程判空时可能拿到引用非 null 但未初始化完的半成品；volatile 写屏障禁止这种重排。（标准 JMM 共八条，正文表格列了六条，另两条是中断规则与终结器规则。）
2. volatile 只解决可见性与有序性（内存屏障），不互斥、不保证原子性；synchronized 通过监视器进入/退出同时解决互斥、可见性、有序性，解锁 happens-before 后续对同一把锁的加锁。所以 volatile 不是“轻量级锁”，临界区该加锁还得加锁。
3. 不可变对象状态全 final，且构造器里 this 不逸出：正确构造的对象，其 final 字段对其他线程可见时保证已初始化完成。发布后任意线程读到的都是完整一致的快照，不需要额外同步；这也是设计并发数据结构时优先“不可变 + AtomicReference 整体替换”，而不是给可变对象层层加锁的原因。

## 8. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 锁语义：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- 运行时内存区域：[JVM-内存模型](开发语言/Java/JVM/JVM-内存模型.md)
- 单例 DCL 落点：[单例](开发语言/Java/设计模式/单例.md)
