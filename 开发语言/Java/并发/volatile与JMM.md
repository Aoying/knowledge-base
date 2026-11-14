---
知识点: volatile与JMM
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-11-14
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

## 7. 可见性延迟与屏障成本定量

本节所有数字均为手算，Python 3 本地复算（见节末“### 复算口径”）。【官方】指 JSR-133 / JLS / JDK 官方文档事实，已用浏览器核实并注明来源链接与核验日期 2026-11-08；【估算】指简化假设下的经验量级，不是官方数字、不是实测值，不许当作基准测试结果引用。原文已有 DCL 三步重排、happens-before 八条、volatile vs synchronized 对比，本节不重复，只往定量方向延伸。

### 7.1 伪共享：同一缓存行里的两个 volatile long

【估算】主流 x86 / ARM 服务器缓存行取 64 字节一行（不是规范数字，是硬件常见值；见边界条件）。一个 `long` 占 8 字节：

- 一行能装 `64 / 8 = 8` 个 `long`。
- 两个相邻的 `volatile long a, b` 若都落在同一行：A 核写 `a` 会使 B 核手里整行失效，B 核再写 `b` 又反过来使 A 核整行失效——变量互不相干，缓存行却互相拖累，这就是伪共享。最坏情况下每次写都触发一次行失效与重取：写 `N` 次 ≈ `N` 次行失效，行流量 ≈ `N × 64` 字节（`N = 1_000_000` 时 ≈ 64 MB 行流量，手算见复算口径）。
- 成本量级【估算】：一次跨核行失效 + 重取约几十到几百个周期，约 10–100 纳秒量级；跨 NUMA 节点 / 跨插槽时可到数百纳秒到微秒级。伪共享的害处不在单次写，而在热循环里每写一次就付一次这个价：每秒百万次写就是百万次失效。

手动填充换算（只算对齐后的理想情形，见边界条件）：

- 想让 `b` 从下一行开始：`a` 占 8 字节，需在中间垫 `64 - 8 = 56` 字节，即 `56 / 8 = 7` 个 `long`。`a`（8 字节）+ 7 个填充 `long`（56 字节）= 64 字节，`b` 恰好落在偏移 64 处。
- 两个各自独占一行的 `long`，最小占 `2 × 64 = 128` 字节；若 `b` 后面还有热字段，还要在 `b` 后面再垫 7 个 `long`，否则下一个字段又会挤进 `b` 的行。

`@Contended` 的换算【官方】：JDK 8 由 JEP 142 引入，Java 9 起位于 `jdk.internal.vm.annotation.Contended`（Java 8 为 `sun.misc.Contended`）。HotSpot 默认 `ContendedPaddingWidth = 128` 字节，即默认垫 2 行（`128 / 64 = 2` 行，`128 / 8 = 16` 个 `long` 当量），多垫的 64 字节是为抵消相邻行预取把伪共享扩到两行的情形；可用 `-XX:ContendedPaddingWidth=64` 改为 1 行。注意它默认只对 JDK 内部类生效，用户代码要用需加 `-XX:-RestrictContended`，且它只是“JVM 自行决定是否加填充”的提示，不是强制保证。

```java
// 手动填充：7 个 long = 56 字节，b 从偏移 64 处起（前提：a 已行对齐，见边界条件）
class Padded {
    volatile long a;
    long p1, p2, p3, p4, p5, p6, p7; // 56 字节填充
    volatile long b;                 // 与 a 不在同一行
}

// 首选写法：让 JVM 垫（用户代码需 -XX:-RestrictContended）
class ContendedCounters {
    @jdk.internal.vm.annotation.Contended
    volatile long a;
    @jdk.internal.vm.annotation.Contended
    volatile long b;
}
```

### 7.2 volatile 读写 vs 普通读写的成本量级

【估算】以下全部是 x86-64、3 GHz、无竞争 / 低竞争热路径下的经验区间，不是官方数字、不是实测值；换 CPU、换 JDK、换竞争程度都会变。换算基准：3 GHz 下 1 周期 ≈ 0.33 纳秒，30 周期 ≈ 10 纳秒，300 周期 ≈ 100 纳秒，3000 周期 ≈ 1 微秒。

| 操作 | 成本量级【估算】 | 为什么 |
|---|---|---|
| 普通读（L1 命中） | 约 1–3 周期，约 0.3–1 纳秒 | 就是一次缓存读 |
| volatile 读 | 约 2–10 周期，约 1–3 纳秒 | 接近普通读；x86 上 volatile 读主要是禁止重排 + 不走寄存器缓存，不额外发锁总线 |
| 普通写（进 store buffer 即返回） | 约 1–3 周期 | 写先入 store buffer，不等刷到缓存 |
| volatile 写（无竞争） | 约 20–100 周期，约 7–33 纳秒 | 约等于一次带 lock 前缀的操作（如 `lock addl`），要等 store buffer 排空 / 拿到行独占权 |
| volatile 写（多核争同一行，即伪共享） | 约 100–300+ 周期，约 33–100+ 纳秒，跨插槽可到微秒级 | 每次写都要等其他核的行失效确认 + 行转移，见 7.1 |

三个定量结论：

1. volatile 读便宜、volatile 写贵：读接近普通读，写比普通写贵约 1 个数量级（20–100 周期 vs 1–3 周期）。读多写少的标志位、配置发布用 volatile 很划算；热循环里每轮都 volatile 写（如计数器）是错的，那是 `AtomicLong` / `LongAdder` 的活。
2. 单次 volatile 写再贵也是纳秒级：每秒写 1 万次，额外开销约 `10_000 × 30 纳秒 = 0.3 毫秒`，可忽略；每秒写 1 千万次，约 `10_000_000 × 30 纳秒 = 0.3 秒`，才值得优化。先算频率再决定要不要优化，别一上来就垫缓存行。
3. volatile 的贵不在“每次都去主内存”：原文冷知识已讲过，它靠屏障 + 缓存一致性协议工作，读仍可命中缓存；贵的是屏障禁掉的优化与一致性流量，不是内存往返本身。

### 7.3 happens-before 传递性链：三线程手算

只用原文第 2 节的规则（程序顺序、volatile 规则、传递性）手算，不引入新规则。场景：

```java
int data = 0;                 // 普通变量
volatile boolean ready = false;
volatile boolean done = false;

// 线程 T1：data = 42; ready = true;
// 线程 T2：if (ready) { int x = data; done = true; }
// 线程 T3：if (done) { int y = data; }
```

逐条推导（hb = happens-before）：

1. T1 内程序顺序：`data = 42` hb `ready = true`（写 volatile 之前）。
2. volatile 规则：T1 的 `ready = true` hb T2 的 `ready` 读（T2 读到 true 的那次）。
3. 传递性：`data = 42` hb T2 的 `ready` 读；T2 内程序顺序又使该读 hb `done = true`，所以 `data = 42` hb T2 的 `done = true`。
4. 再接一段 volatile 规则 + 传递性：T2 的 `done = true` hb T3 的 `done` 读，T3 内该读 hb `data` 读，所以 `data = 42` hb T3 的 `data` 读。
5. 结论：T3 读到 `done == true` 时，保证读到 `data == 42`；T2 读到 `ready == true` 时，也保证读到 `data == 42`。

哪些读不保证：

- T3 不读 `done`、直接读 `data`：与 T1 的写之间没有任何 hb 链，可能读到 0，也可能读到 42——无保证，不是“大概率 42”。
- T2 读到 `ready == true`，但 T3 只读了 T2 写的一个普通 `boolean finished`（非 volatile、无锁）：`finished` 的写读之间没有 hb 边，链在 T2 → T3 处断了，T3 读 `data` 不保证是 42。发布通道和读取通道必须配对，链上任何一环换成普通变量，整条链就断了。
- T2 读到 `ready == false` 就直接读 `data`：这次读发生在 hb 链建立之前，同样无保证。

### 7.4 store buffer / invalidate queue：可见性延迟量级

【估算】以下延迟是“写已发出、其他核何时能读到”的经验区间，不是 JMM 的保证——JMM 只保证有 hb 关系时读到，不保证多快读到；没有 hb 关系时，规范不给任何可见性时限。

- 同核 / 同插槽、无竞争：store buffer 排空 + 行状态转移约几十个周期，约 10–100 纳秒。
- 跨核争用同一行（伪共享场景）：等 invalidate 确认 + 行转移，约 100–300 周期，约 33–100 纳秒。
- 跨插槽 / 跨 NUMA 节点、队列已积压：约 1000–3000+ 周期，约 0.3–1+ 微秒；极端积压下可到数微秒。
- 换算口径：延迟（纳秒）≈ 周期数 / 3（3 GHz 下，见复算口径）。

所以“另一个线程 1 微秒后还没看到”在规范上完全合法，靠 `sleep` 等可见性是赌时序；要保证可见性只能建 hb 边（volatile / 锁 / start-join / 线程安全队列），不能靠“等一会儿”。

### 7.5 边界条件

1. 缓存行不是 64 字节放之四海而皆准【估算 + 官方背景】：x86-64 常见 64 字节，但部分 ARM / Apple Silicon 为 128 字节，个别 Power 机器也是 128 字节。按 64 字节垫 7 个 `long`，在 128 字节行上两个变量仍可能同行；`@Contended` 默认垫 128 字节正是为此留了余量，跨平台代码优先用它而不是手算 7 个 `long`。
2. 手动填充的 7 个 `long` 只有在 `a` 行对齐时才成立：对象头（压缩指针下约 12 字节，按 8 字节对齐后占 16 字节）会让第一个字段的偏移不是 64 的倍数，HotSpot 还可能重排字段顺序、把填充字段优化掉（未被读写的 `long` 可能被 JIT 判定为死代码消除）。手动填充是“希望如此”，不是保证；要验证只能用 JOL 看实际布局。
3. 只有“多核同时高频写不同变量”才配谈伪共享：两个变量只读不写不会互相 invalidate；写频率很低（如每秒几次的配置计数）时，失效成本摊薄到可忽略；单核跑、或两个变量总被同一线程写，也不存在伪共享。先确认是多核热写，再垫。
4. `@Contended` 不是免费午餐：默认每字段多占 128 字节，数组 / 大量小对象上用它会把内存撑大数倍，且默认不对用户代码生效（需 `-XX:-RestrictContended`）；冷字段、只读字段不要加，只给实测确认的热竞争字段加。
5. 本节所有周期数、纳秒数都是【估算】：不同 CPU 型号、频率、NUMA 拓扑、JDK 版本下都会变；要精确数字只能在目标机器上用 JMH 实测，本节数字只用于“哪个操作贵一个数量级、该不该优化”的量级判断。

### 复算口径

手算过程用 Python 3 本地复算，输出如下（`python3` 直接运行，无第三方依赖）：

```python
cache_line = 64      # 字节，主流 x86 / ARM 常见值【估算】
long_bytes = 8       # long 占 8 字节
print("longs_per_line =", cache_line // long_bytes)          # 8
print("pad_longs_between =", cache_line // long_bytes - 1)   # 7
print("pad_bytes =", 7 * long_bytes)                         # 56
print("second_long_offset =", long_bytes + 56)               # 64，恰为下一行起点
print("two_padded_longs =", 2 * cache_line)                  # 128 字节
print("contended_default_lines =", 128 // 64)                # 2 行
print("contended_default_longs =", 128 // 8)                 # 16 个 long 当量
for c in [1, 3, 10, 30, 100, 300, 1000, 3000]:
    print(c, "cycles @3GHz =", round(c / 3, 1), "ns")
# 复算输出：longs_per_line = 8；pad_longs_between = 7；pad_bytes = 56；
# second_long_offset = 64；two_padded_longs = 128；
# contended_default_lines = 2；contended_default_longs = 16；
# 1 / 3 / 10 / 30 / 100 / 300 / 1000 / 3000 周期 @3GHz
# = 0.3 / 1.0 / 3.3 / 10.0 / 33.3 / 100.0 / 333.3 / 1000.0 纳秒
# 伪共享行流量：N = 1_000_000 次写 -> N * 64 字节 = 64_000_000 字节 ≈ 64 MB
```

来源（核验日期 2026-11-08）：

- 【官方】JLS 17.4.5 happens-before 规则、volatile 写 hb 随后读同一变量：`java.util.concurrent` 包文档（JDK 17，引用 JLS 17.4.5）——https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html
- 【官方】JLS 17.7 非 volatile long / double 单次写按两次 32 位写处理、volatile long / double 读写恒原子：https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.7（主代理实读核验）
- 【官方】JEP 142 引入 `@Contended`、默认 128 字节填充、为抵消相邻行预取而取两倍行宽、需 `-XX:-RestrictContended` 才对用户代码生效：OpenJDK hotspot-dev 邮件（JEP 142 实现说明）——https://mail.openjdk.org/pipermail/hotspot-dev/2012-November/007309.html ；另见 https://www.baeldung.com/java-false-sharing-contended 与 https://jenkov.com/tutorials/java-concurrency/false-sharing.html
- 【估算】本节全部周期数、纳秒数、延迟区间均为经验量级估算，不是官方数字、不是实测值；缓存行 64 字节为 x86 / ARM 常见值，128 字节行见边界条件第 1 条。

## 原理剖析

本节只把原文已有章节给出的结论拆开讲清机制：不引入新概念，不补新数字，数字一律复用 §7 已核验的定量算例；凡原文没展开的点，文中明确标注，不作定论。

### happens-before：从“能看到”到“一定能看到”

原文 §2 给出的判断方法是：若 A happens-before B，则 A 的结果对 B 一定可见；反过来，没有 happens-before 关系，就没有可见性保证。这一点要先拆开：happens-before 回答的是“谁能看到谁的写”，而不是“谁先执行”。原文 §2 的表格只列出规则本身，规则背后的执行顺序仍由调度决定，原文未展开调度与 happens-before 的关系，此处不作定论。

把 §2 的规则串起来看，关键在传递性。原文 §7.3 用三线程手算示范了这条链：T1 内程序顺序使 `data = 42` happens-before `ready = true`；volatile 规则使 T1 的 `ready = true` happens-before T2 读到 `ready` 的那次读；传递性再把这两段接起来，得到 `data = 42` happens-before T2 的 `ready` 读。T2 内程序顺序又使该读 happens-before `done = true`，于是 `data = 42` 也 happens-before T2 的 `done = true`；再接一段 volatile 规则与传递性，`data = 42` 最终 happens-before T3 读 `data` 的那次读。所以 T3 读到 `done == true` 时，保证读到 `data == 42`；T2 读到 `ready == true` 时，也保证读到 `data == 42`。

链断在哪里，保证就断在哪里。原文 §7.3 列出的三种不保证的情形，机制是同一个：T3 不读 `done`、直接读 `data`，与 T1 的写之间没有任何 happens-before 链；T2 写的是一个普通变量而不是 volatile 变量，链在 T2 到 T3 之间断了；T2 读到 `ready == false` 就直接读 `data`，这次读发生在 happens-before 链建立之前。三者都不是“读得不够快”，而是链本身不存在。原文没有展开“读到 false 的那次读与之后某次读到 true 的读之间是什么关系”，原文未展开，此处不作定论。

### volatile 可见性：写 volatile 标志时，之前写过的数据一并可见

原文 §2 的推论是 volatile 的“顺带刷新”：写 volatile 变量时，它之前所有普通变量的修改会一并刷给后续读到该 volatile 变量的线程。拆开讲，发布模式“先写数据、再写 volatile 标志位”成立，靠的不是标志位本身有什么特殊内容，而是两段 happens-before 接在了一起：同一线程内程序顺序使“写数据” happens-before “写 volatile 标志”；volatile 规则使“写 volatile 标志” happens-before 另一线程“读 volatile 标志”的那次读；传递性把“写数据”与“读数据”接起来，读方才能保证看到数据。

这个保证是有条件的，条件就是读方必须读到那个 volatile 变量。原文 §2 与 §7.3 都只说“读到该 volatile 变量的线程”，没有说“所有线程”。一个线程根本没读这个标志、直接去读数据，它与写方之间没有这条链，看到的仍可能是旧值。原文没有展开写 volatile 变量时底层具体如何把之前普通变量的修改刷出去、以及读方具体在何时完成同步，原文未展开，此处不作定论。

可见性保证也不等于“立刻可见”。原文 §7.4 明确区分了两件事：JMM 只保证有 happens-before 关系时读到，不保证多快读到；没有 happens-before 关系时，规范不给任何可见性时限。§7.4 给出的延迟量级均为【估算】：同核或同插槽、无竞争时约几十个周期，约 10–100 纳秒；跨核争用同一行时约 100–300 周期，约 33–100 纳秒；跨插槽或跨 NUMA 节点、队列已积压时约 1000–3000+ 周期，约 0.3–1+ 微秒，极端积压下可到数微秒（出处：§7.4）。所以“另一个线程 1 微秒后还没看到”在规范上完全合法；要保证可见性只能建 happens-before 边，不能靠“等一会儿”。原文没有展开 store buffer 排空与 invalidate 队列处理的具体触发条件，原文未展开，此处不作定论。

### volatile 有序性与 DCL：禁止重排修的是“半成品可见”

原文 §3 说，重排来自编译器、JIT 与 CPU，单线程语义不变的前提下它们可以任意调整独立指令的顺序；多线程下没有同步手段时，别人看到的顺序可能就是乱的。volatile 的第二个作用是禁止特定重排，靠插入内存屏障实现。原文只说“插入内存屏障”“禁止特定重排”，没有展开屏障具体插在读写操作的哪一侧、不同读写组合对应哪种屏障，原文未展开，此处不作定论。

DCL 是这个机制最直接的落点。原文 §4 说，`new` 分三步：分配内存、调用构造初始化、引用指向内存；没有 volatile 时后两步可重排，另一个线程在第一次判空时可能拿到“引用非 null 但对象未初始化完”的半成品。拆开讲：第一次判空读到非 null，只能说明引用赋值这一步已经发生了，不能说明构造初始化已经完成；因为重排允许“引用指向内存”先于“调用构造初始化”被其他线程看到，判空通过与对象可用之间没有 happens-before 保证。volatile 写屏障禁止这种重排，写 volatile 字段之前的操作不会被排到写之后，读方读到该 volatile 字段时，按 §2 的 volatile 规则与传递性，写之前已完成的初始化对读方可见。这也是原文 §4 说的“把 DCL 从‘看起来对’修成‘真的对’”的意思。

同一段逻辑也解释了原文 §3 的 `data` 与 `ready` 例子：线程 A 写 `data = 42` 再写 `ready = true`，volatile 写禁止前面的写被排到它后面；线程 B 读到 `ready` 为 true 时能读到 42，靠的是 happens-before 传递性加顺带刷新，而不是靠 `ready` 这个布尔值本身携带了 `data`。原文没有展开 JIT 把循环里的读优化成只读一次寄存器的具体判定条件，原文未展开，此处不作定论；原文 §3 只给出“没有 volatile 时，JIT 可能把读优化成只读一次寄存器，永远看不到新值”这一结论。

还要守住原文 §3 与 §6 的边界：`volatile int count; count++` 依然会丢更新。`count++` 是读、加、写三步，volatile 只保证每一步读到最新值，不保证三步不可分割；计数场景用原文 §3 指向的 CAS 与原子类或加锁。把 volatile 当“轻量级锁”是原文 §6 列出的误区：它不互斥，临界区该加锁还得加锁。

### final 的内存语义：构造完成之前不发布，发布之后读到的就是完整的

原文 §5 的结论是：正确构造的对象，其 final 字段对其他线程可见时保证已初始化完成；前提是构造器里不能让 `this` 逸出，否则该保证作废。拆开讲，这里的“正确构造”与“可见时”是两个条件，缺一不可：对象必须先由构造器完整构造出来，再通过一条有 happens-before 保证的通道发布出去，读方读到的 final 字段才保证是构造完成后的值。原文没有展开 final 字段语义在 happens-before 规则表里对应哪一条、以及发布通道具体有哪些，原文未展开，此处不作定论。

这也是原文 §5 说不可变对象天然线程安全的根源：状态全 final 加不泄漏 `this`，发布后任意线程读到的都是完整一致的快照。反过来，构造器里让 `this` 逸出，等于是把“还没构造完的对象”提前交给了别的线程，别的线程可能读到半初始化的对象，final 字段的保证随之作废。原文没有展开 `this` 逸出的全部姿势，原文未展开，此处不作定论；原文只在 §5 给出“构造器里不能让 `this` 逸出”这一条。

## 实战案例

### 案例一：停止标志改了，循环就是停不下来

**现象：** 复用原文 §3 的写法，一个工作线程在 `while (running)` 里干活，另一个线程调用 `stop()` 把开关置为 `false`，但工作线程一直不停。

```java
class Worker {
    private volatile boolean running = true;   // 停止开关必须 volatile
    void run() {
        while (running) { /* 干活 */ }          // 没有 volatile 时，JIT 可能把读优化成
    }                                           // 只读一次寄存器，永远看不到新值
    void stop() { running = false; }
}
```

**排查：** 先看字段本身是不是 volatile。原文 §6 的误区说，用普通 boolean 做跨线程开关必翻车，只是时间问题；没有 volatile 时，`stop()` 的写与工作线程的读之间没有 happens-before 边，工作线程可能永远读到旧值。原文 §3 给出的机制是，JIT 可能把读优化成只读一次寄存器，永远看不到新值；原文没有展开 JIT 这样优化的具体判定条件，原文未展开，此处不作定论。其次看读线程有没有把旧值缓存在局部变量里：读线程一旦把标志取到局部变量后一直用它，再怎么刷新字段也看不到，这一步的排查方向与字段本身是不是 volatile 不同，修的方向也不同。最后不要用“等一会儿”来验证：原文 §7.4 说，没有 happens-before 关系时规范不给任何可见性时限，“另一个线程 1 微秒后还没看到”在规范上完全合法；同核或同插槽、无竞争时延迟约 10–100 纳秒，跨插槽或跨 NUMA 节点、队列已积压时约 0.3–1+ 微秒、极端积压下可到数微秒，这些数字均为【估算】（出处：§7.4），只能说明延迟量级，不能当作“睡够这么久就一定能看到”的保证。

**结论：** 停止开关字段加 volatile，让 `stop()` 的写 happens-before 工作线程随后对同一变量的读；同时确认读线程每次循环都重新读字段，而不是读一次后一直用局部变量里的旧值。原文 §3 的 `Worker` 写法就是这个结论对应的代码。成本上不用担心读标志本身：原文 §7.2 的【估算】是 volatile 读约 2–10 周期，约 1–3 纳秒，接近普通读（出处：§7.2）；贵的是 volatile 写，但停止标志只写一次，不在热循环里反复写。

### 案例二：两个互不相干的 volatile 计数器，越写越慢

**现象：** 复用原文 §7.1 的设定，两个相邻的 `volatile long a, b` 被不同线程高频写，变量之间互不相干，但写操作越写越慢。

```java
// 手动填充：7 个 long = 56 字节，b 从偏移 64 处起（前提：a 已行对齐，见边界条件）
class Padded {
    volatile long a;
    long p1, p2, p3, p4, p5, p6, p7; // 56 字节填充
    volatile long b;                 // 与 a 不在同一行
}

// 首选写法：让 JVM 垫（用户代码需 -XX:-RestrictContended）
class ContendedCounters {
    @jdk.internal.vm.annotation.Contended
    volatile long a;
    @jdk.internal.vm.annotation.Contended
    volatile long b;
}
```

**排查：** 先确认是不是伪共享，而不是先怀疑 volatile 本身。原文 §7.1 的机制是：主流 x86 / ARM 服务器缓存行取 64 字节一行【估算】，一个 `long` 占 8 字节，一行能装 8 个 `long`；两个相邻的 `volatile long a, b` 若都落在同一行，A 核写 `a` 会使 B 核手里整行失效，B 核再写 `b` 又反过来使 A 核整行失效，变量互不相干，缓存行却互相拖累。最坏情况下每次写都触发一次行失效与重取：写 `N` 次约 `N` 次行失效，行流量约 `N × 64` 字节，`N = 1_000_000` 时约 64 MB 行流量（出处：§7.1）。再看成本量级：原文 §7.1 与 §7.2 的【估算】是，一次跨核行失效与重取约几十到几百个周期，约 10–100 纳秒量级，跨 NUMA 节点或跨插槽时可到数百纳秒到微秒级；volatile 写在多核争同一行时约 100–300+ 周期，约 33–100+ 纳秒，跨插槽可到微秒级（出处：§7.1、§7.2）。还要先排除不成立的前提：原文 §7.5 说，只有“多核同时高频写不同变量”才配谈伪共享，两个变量只读不写不会互相 invalidate，写频率很低时失效成本摊薄到可忽略，单核跑、或两个变量总被同一线程写，也不存在伪共享。

**结论：** 确认是多核热写不同变量导致的伪共享后，把两个变量分到不同缓存行。按原文 §7.1 的换算，`a` 占 8 字节，中间垫 `64 - 8 = 56` 字节即 7 个 `long`，`b` 恰好落在偏移 64 处；两个各自独占一行的 `long` 最小占 `2 × 64 = 128` 字节（出处：§7.1）。跨平台优先用原文 §7.1 的 `@Contended` 写法：JDK 8 由 JEP 142 引入，Java 9 起位于 `jdk.internal.vm.annotation.Contended`，HotSpot 默认 `ContendedPaddingWidth = 128` 字节，即默认垫 2 行（`128 / 64 = 2` 行，`128 / 8 = 16` 个 `long` 当量），用户代码使用需加 `-XX:-RestrictContended`（出处：§7.1）。原文 §7.5 的边界条件要一并记住：手动垫 7 个 `long` 只有在 `a` 行对齐时才成立，对象头与字段重排、JIT 消除都可能让它失效，手动填充是“希望如此”，不是保证；`@Contended` 默认每字段多占 128 字节，冷字段、只读字段不要加，只给实测确认的热竞争字段加。原文没有展开如何实测确认某个字段确实是热竞争字段，原文未展开，此处不作定论。

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

## 8. 面试与自测

- happens-before 八条规则说出常用的五条，并用它解释 DCL 为什么需要 volatile。
- volatile 与 synchronized 在可见性实现上的差别是什么？
- 为什么不可变对象不需要同步也能安全发布？

- 两个 `volatile long` 落在同一 64 字节缓存行里会有什么问题？要把它们分到不同行，中间垫几个 `long`？`@Contended` 默认垫多少字节、用户代码使用时还缺什么启动参数？

### 参考答案

1. 常用的五条：程序顺序（同线程内书写在前 happens-before 在后）、锁规则（解锁 happens-before 随后对同一把锁的加锁）、volatile 规则（volatile 写 happens-before 随后对同一变量的读）、线程启动（start() 之前的操作 happens-before 新线程内的操作）、线程终止（线程内操作 happens-before 其他线程 join() 返回后），再加传递性串联。DCL 需要 volatile 的原因：new 分三步（分配内存、构造初始化、引用指向内存），无 volatile 时后两步可重排，另一线程判空时可能拿到引用非 null 但未初始化完的半成品；volatile 写屏障禁止这种重排。（标准 JMM 共八条，正文表格列了六条，另两条是中断规则与终结器规则。）
2. volatile 只解决可见性与有序性（内存屏障），不互斥、不保证原子性；synchronized 通过监视器进入/退出同时解决互斥、可见性、有序性，解锁 happens-before 后续对同一把锁的加锁。所以 volatile 不是“轻量级锁”，临界区该加锁还得加锁。
3. 不可变对象状态全 final，且构造器里 this 不逸出：正确构造的对象，其 final 字段对其他线程可见时保证已初始化完成。发布后任意线程读到的都是完整一致的快照，不需要额外同步；这也是设计并发数据结构时优先“不可变 + AtomicReference 整体替换”，而不是给可变对象层层加锁的原因。

4. 会伪共享：两核写不同变量却共享同一缓存行，每次写都使对方整行失效并重取，热循环下每次写都付几十到几百周期的跨核失效成本。分行换算：64 字节一行装 8 个 `long`，第一个占 8 字节，中间垫 `64 - 8 = 56` 字节即 7 个 `long`，第二个恰好落在偏移 64 处；两个独占一行的 `long` 最少占 128 字节。`@Contended`（JEP 142，JDK 8 引入，Java 9 起在 `jdk.internal.vm.annotation`）默认垫 128 字节（2 行，防相邻行预取），可用 `-XX:ContendedPaddingWidth` 改；默认只对 JDK 内部类生效，用户代码需加 `-XX:-RestrictContended`。注意手动垫 7 个 `long` 只有在首字段行对齐且未被 JIT 消除时才成立，跨平台优先用 `@Contended`。

## 9. 关联知识

### 前置知识

- [Java-并发编程](开发语言/Java/并发/Java-并发编程.md)：并发全景与汇总入口，先建立并发问题域再读本篇的可见性与有序性（引用原文 §1、§9）。
- [synchronized与锁](开发语言/Java/并发/synchronized与锁.md)：锁规则是 happens-before 的另一条主线，与 volatile 规则对照着理解可见性（引用原文 §2、§9）。
- [JVM-内存模型](开发语言/Java/JVM/JVM-内存模型.md)：运行时数据区与对象内存布局，先分清它与 JMM 的区别再读本篇（引用原文 §1、§9）。
- [单例](开发语言/Java/设计模式/单例.md)：DCL 在单例里的落点，与本篇 §4 讲的是同一个坑（引用原文 §4、§9）。
- [CAS与原子类](开发语言/Java/并发/CAS与原子类.md)：volatile 不解决原子性，计数与复合操作要走 CAS 或加锁（引用原文 §3）。

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 锁语义：[synchronized与锁](开发语言/Java/并发/synchronized与锁.md)
- 运行时内存区域：[JVM-内存模型](开发语言/Java/JVM/JVM-内存模型.md)
- 单例 DCL 落点：[单例](开发语言/Java/设计模式/单例.md)
