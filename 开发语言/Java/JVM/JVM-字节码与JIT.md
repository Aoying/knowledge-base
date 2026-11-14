---
知识点: JVM 字节码与 JIT
层级: 进阶
适用版本: JDK 8+（分层编译 JDK 7 引入、JDK 8 默认开启）
更新日期: 2026-11-14
关联:
  - JVM-内存模型
  - JVM-类加载
  - JVM-GC调优
  - JVM-故障排查
标签:
  - Java
  - JVM
  - 原理
---

# JVM 字节码与 JIT

## 1. 核心概念

- Java 源码经 `javac` 编译为平台无关的字节码（`.class`），JVM 先解释执行，运行期把**热点代码**由 JIT 编译成本地机器码——「解释 + 编译」混合执行是 HotSpot 的基本形态。
- `javac` 几乎不做优化（只有常量折叠等少数动作），真正的优化全在 JIT，且基于运行期的真实调用数据，所以同一份字节码在不同负载下表现可以差很多。
- 本篇解决三类问题：看懂 `javap` 反汇编（如装箱、字符串拼接的真实开销）、理解 JIT 行为（为什么要预热、基准为何不准）、排查 JIT 故障（代码缓存满、去优化抖动）。

## 2. class 文件结构

class 文件是严格有序的二进制格式，与语言无关（Kotlin、Groovy 最终也编译成它），依次为：魔数 `0xCAFEBABE`（固定 4 字节）→ 版本号（JDK 8 主版本 52、JDK 17 为 61，高版本 class 不能跑低版本 JVM）→ **常量池**（类名、方法名、字符串等字面量与符号引用，指令靠索引引用）→ 访问标志 → 类/父类/接口索引 → 字段表 → 方法表（字节码在方法的 Code 属性里）→ 属性表（调试信息、注解等）。

- 方法执行模型：每个方法有一个**操作数栈**和一张**局部变量表**。指令大多只操作栈顶：`iload_1` 把局部变量 1 压栈，`iadd` 弹两个数相加再压回，`istore_2` 弹出存入局部变量 2。没有寄存器概念，这是字节码可移植的代价，也是 JIT 要消除的开销。

## 3. 常用指令

| 类别 | 代表指令 | 说明 |
| --- | --- | --- |
| 加载 / 存储 / 运算 | `iload`、`aload`、`istore`、`iadd`、`iinc` | 在局部变量表与操作数栈间搬运与计算；`long`/`double` 占两个槽位 |
| 类型转换 | `i2l`、`checkcast`、`instanceof` | `checkcast` 失败抛 `ClassCastException` |
| 对象操作 | `new`、`getfield`、`putfield`、`getstatic` | `new` 只分配内存，构造走 `invokespecial <init>` |
| 方法调用 | 五种 `invoke*` | 理解多态与 Lambda 的关键，见下 |
| 控制转移 | `ifeq`、`goto`、`tableswitch`、`lookupswitch` | `switch` 稠密值用前者、稀疏值用后者 |
| 异常 | `athrow` + 异常表 | try-catch 靠 Code 属性里的异常表按 pc 范围匹配，不靠跳转指令 |

调用指令的区别最值得记牢：`invokestatic` 静态方法、编译期绑定；`invokespecial` 构造器 / private / 父类方法；`invokevirtual` 普通虚方法、按接收者实际类型经虚方法表分派；`invokeinterface` 接口方法、分派略慢于 virtual；`invokedynamic`（JDK 7）把调用逻辑延迟到运行期决定，Lambda、方法引用、JDK 9+ 的字符串拼接都靠它。

## 4. javap 读法

`javap -c -p Demo.class` 看指令（含 private 成员）；`-v` 看常量池与属性全量；`-s` 附方法描述符。

源码：

```java
public int add(int a, int b) { return a + b; }
```

对应字节码：

```text
0: iload_1        // a 压栈（槽 0 是 this）
1: iload_2        // b 压栈
2: iadd           // 相加
3: ireturn        // 返回栈顶 int
```

实战价值：怀疑有隐性开销时直接验证——如循环里 `Integer` 装箱会出现 `invokestatic Integer.valueOf`；字符串 `+` 拼接在 JDK 8 是 `StringBuilder` 链、JDK 9+ 是一条 `invokedynamic`。不看字节码只能猜。

## 5. JIT 编译

### 5.1 热点探测与触发

- HotSpot 用两个计数器统计：**方法调用计数器**与**回边计数器**（循环回跳时累加），之和达阈值即判定热点、触发编译；计数随时间衰减，避免偶发高频误判。
- 编译由后台线程异步进行，不阻塞业务线程；完成前继续解释执行。循环先热起来的场景走 **OSR（栈上替换）**：方法还在栈上就把当前帧换成编译版本，不等下一次调用。

### 5.2 分层编译

| 层级 | 形态 | 特点 |
| --- | --- | --- |
| Tier 0 | 解释执行 | 启动即用，慢 |
| Tier 1–3 | C1 编译 | 编译快、优化浅；不同层带不同程度的性能剖析 |
| Tier 4 | C2 编译 | 编译慢、优化深，长期热点才值得 |

- 典型路径：解释 → C1（带剖析）→ 收集调用频率、分支走向、接收者类型 → C2 激进优化。JDK 8 起默认开启，不要按老资料手动加 `-XX:-TieredCompilation` 关掉，通常让启动期更差。C1 适合短生命周期、启动敏感场景；C2 适合长跑服务追求峰值吞吐。

### 5.3 C2 的核心优化

| 优化 | 做什么 |
| --- | --- |
| 方法内联 | 被调用方法体嵌入调用处，消除调用开销并打开后续优化空间；方法不能太大，虚方法要先去虚化 |
| 去虚化 | 剖析显示接收者只有一个实现类时按单态直接内联，插类型守卫，猜错就去优化 |
| 逃逸分析 | 分析对象引用是否逃出方法 / 线程，是下面三项的前提 |
| 标量替换 | 不逃逸对象拆成标量放栈上 / 寄存器，对象根本不分配、随栈帧消失，收益最大 |
| 同步消除 | 对不逃逸对象上的 `synchronized` 直接去锁，如局部 `StringBuffer` |
| 循环优化 | 循环展开、范围检查消除、循环不变量外提 |

- **去优化（Deoptimization）**：投机假设被打破时（如新加载了子类实现、分支走向突变），已编译代码作废、退回解释执行重新剖析。偶发正常，高频会造成性能抖动，看 `PrintCompilation` 里是否反复出现 `made not entrant`。

## 6. JIT 编译阈值与内联预算定量

本节把 §5 的“热点→编译”讲成数字。版本口径：默认 64 位 HotSpot、JDK 8–21，分层编译开启（JDK 8 起默认开启）；凡是写了“JDK 8 / JDK 11+ / JDK 17 / JDK 21”的地方，按那个版本理解，其余数字在 JDK 8 到 JDK 21 间未变。所有默认值均以 `java -XX:+PrintFlagsFinal` 在目标 JDK 上实测为准，本节数字核验日期 2026-11-08，来源见每表下方。

### 6.1 分层编译阈值：调用次数 + 回边次数

分层编译不看时间、只看计数器：方法调用次数 `i` 与循环回边次数 `b`（见 §5.1）。从解释执行升到 Tier 3（C1 带剖析）、再升到 Tier 4（C2），判定谓词是同一个（`s` 是编译队列长度带来的放大系数，队列空时 `s = 1`）：

```text
i > TierXInvocationThreshold × s
  或 (i > TierXMinInvocationThreshold × s 且 i + b > TierXCompileThreshold × s)
```

| 参数 | Tier 3（0 → 3） | Tier 4（3 → 4） | 含义 |
| --- | --- | --- | --- |
| `TierXInvocationThreshold` | 200 | 5000 | 光靠调用次数就升层 |
| `TierXMinInvocationThreshold` | 100 | 600 | 走“调用 + 回边”混合口径时调用次数的下限 |
| `TierXCompileThreshold` | 2000 | 15000 | 混合口径里 `调用次数 + 回边次数` 的门槛 |
| `TierXBackEdgeThreshold` | 60000 | 40000 | OSR 口径，见 6.2 |

来源：OpenJDK 源码 `src/hotspot/share/runtime/globals.hpp`（Tier3/Tier4 四项默认值）；判定谓词见 `src/hotspot/share/compiler/tieredThresholdPolicy.hpp`；数值与公式另见微软 OpenJDK 团队《How Tiered Compilation works in OpenJDK》。

手算例（Tier 3 混合口径）：某方法每次调用跑 20 次循环回边。调用 150 次时 `i = 150`、`b = 150 × 20 = 3000`，`i + b = 3150`。`100 < 150 < 200`，调用次数没到 200，但 `150 > 100 且 3150 > 2000`，混合口径成立，升到 Tier 3。纯调用、无循环的方法则要攒满 200 次调用才升 Tier 3。

手算例（Tier 4 混合口径）：同一方法升到 Tier 3 后继续跑，在 Tier 3 代码里累计 `i' = 700` 次调用、`b' = 15000` 次回边，`700 > 600 且 700 + 15000 = 15700 > 15000`，升到 Tier 4。没有循环的方法要攒满 5000 次 Tier 3 调用才升 Tier 4。还有一条捷径：C1 剖析后发现方法里没有任何值得剖析的东西（无分支、无调用），会直接从 Tier 3 升到 Tier 4，不等阈值。

### 6.2 OSR 触发口径：回边计数器说了算

只被调用一次、但内部循环跑很久的方法（如 `main` 里的长循环）等不到“下一次调用”，靠 OSR（栈上替换，见 §5.1）在循环跑到一半时直接换上编译版本。口径只有回边次数 `b`：

| 场景 | OSR 阈值 | 说明 |
| --- | --- | --- |
| 分层编译，OSR 到 Tier 3 | `b > 60000`（`Tier3BackEdgeThreshold`） | 循环回边满 60000 次 |
| 分层编译，OSR 到 Tier 4 | `b' > 40000`（`Tier4BackEdgeThreshold`） | 在 Tier 3 代码里回边再满 40000 次 |
| 关掉分层编译（`-XX:-TieredCompilation`） | `b > 14000` | 见下式，非分层模式才用这个口径 |

非分层模式的 OSR 阈值是算出来的，不是直接配的：`CompileThreshold × OnStackReplacePercentage / 100 = 10000 × 140 / 100 = 14000`。其中 `CompileThreshold` 默认 10000（仅分层关闭时起作用，分层开启时方法升层不看它），`OnStackReplacePercentage` 默认 140。分层开启时这两个参数不参与 OSR 判定，别拿 14000 去套分层编译的 JVM。

来源：`TierXBackEdgeThreshold` 默认值见 `globals.hpp`；`CompileThreshold = 10000`、`OnStackReplacePercentage = 140` 见 OpenJDK 源码 `src/hotspot/share/runtime/globals.hpp` 与平台相关 `c2_globals_*.hpp`；分层下 OSR 口径另见 JDK 17.0.16 / JDK 21.0.8 实测验证的可视化分析。

### 6.3 内联预算：字节码论大小，机器码论“已编译”

内联的预算分两套尺子：一套量被调方法的**字节码**大小，一套量它**已编译成的机器码**大小。超过预算的调用点不内联，但方法本身照常被 JIT 编译——丢的只是跨方法边界的优化（逃逸分析、常量传播等），不是编译本身。

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `MaxInlineSize` | 35（字节码字节数） | 冷调用点：被调方法字节码超过 35 字节就不内联 |
| `FreqInlineSize` | 325（字节码字节数，x86_64） | 热调用点：上限放宽到 325 字节 |
| `MaxTrivialSize` | 6（字节码字节数） | ≤ 6 字节的极小方法（如简单 getter）无条件优先内联 |
| `MaxInlineLevel` | 9（JDK 8）；15（2020 年后源码） | 嵌套内联最多几层，见 6.5 的版本说明 |
| `MaxRecursiveInlineLevel` | 1 | 递归调用最多内联 1 层 |
| `InlineSmallCode` | 2000（x86）；2500（SPARC / AArch64），单位是已编译机器码字节数 | 被调方法已有编译版本且机器码小于该值时才内联；大于该值判 "already compiled into a big method"，中等的按 `InlineSmallCode / 4`（x86 即 500）判 |
| `DesiredMethodLimit` | 8000（字节码字节数） | 内联后调用方方法体超过 8000 字节码就停止继续内联 |
| `MinInliningThreshold` | 250（JDK 17） | JDK 17 要求被调方法至少跑过 250 次才考虑内联；JDK 21 已取消这条限制 |

来源：`MaxInlineSize = 35`、`MaxTrivialSize = 6`、`MaxRecursiveInlineLevel = 1` 见 OpenJDK 源码 `src/hotspot/share/opto/c2_globals.hpp`；`MaxInlineLevel` 见同文件（2020 年源码为 15）与 JDK 8 实测 `PrintFlagsFinal`（2016 年 JITWatch 讲义，值为 9）；`FreqInlineSize = 325`（x86_64 平台相关值）与 `InlineSmallCode` 的 2000 / 2500 见 `src/hotspot/share/runtime/tieredThresholdPolicy.cpp`（分层策略启动时按平台改写，x86 取 2000，SPARC / AArch64 取 2500）；`DesiredMethodLimit = 8000` 见 `c2_globals.hpp`；`MinInliningThreshold = 250`（JDK 17 有、JDK 21 已取消）见 JDK 17.0.16 / JDK 21.0.8 实测验证的可视化分析。

手算例（与小技巧节呼应）：某方法字节码 40 字节。冷调用时 `40 > 35`，超过 `MaxInlineSize`，判 "too big"，不内联——只差 5 字节也进不去。跑热后调用点变热，改按 `FreqInlineSize` 算：`40 ≤ 325`，还剩 `325 − 40 = 285` 字节余量，可以内联；热门槛是冷门槛的 `325 / 35 ≈ 9.29` 倍。但“可以”不等于“一定”：还得同时满足内联深度 ≤ `MaxInlineLevel`、内联后方法体 ≤ `DesiredMethodLimit = 8000` 字节码、JDK 17 下被调方法已跑满 `MinInliningThreshold = 250` 次。`javap -c` 看到的就是这个字节数，先量大小再谈为什么没内联。

### 6.4 CodeCache：240MB 与三段划分

编译产物存在 CodeCache（非堆独立区域，见 §8）。默认大小分两种情况：

| 情况 | `ReservedCodeCacheSize` 默认值 |
| --- | --- |
| 分层编译开启（默认，JDK 8 起） | 240MB（即 48MB × 5，`240 × 1024 × 1024 = 251658240` 字节 = 245760KB） |
| 分层编译关闭（`-XX:-TieredCompilation`） | 48MB（Server VM；Client VM 为 32MB） |

来源：Oracle《Codecache Tuning》（JDK 8 文档，`ReservedCodeCacheSize` 默认 32M / 48M）；分层开启时 240MB 见 JEP 197 与 `src/hotspot/share/code/codeCache.hpp` 注释（分段默认开启条件：分层编译开启且 `ReservedCodeCacheSize ≥ 240MB`）。

JDK 9 起（JEP 197）CodeCache 默认分段成三个堆，大小按“非方法段先取固定值、剩余两段对半分”算出：

| 段 | 放什么 | 默认大小（按 240MB 算） |
| --- | --- | --- |
| non-nmethods（`NonNMethodCodeHeapSize`） | 解释器、适配器、运行时桩等非方法代码 | 约 5MB（另加编译器缓冲，实际略大） |
| profiled nmethods（`ProfiledCodeHeapSize`） | C1 编译的带剖析代码，生命周期短 | 约 117.5MB |
| non-profiled nmethods（`NonProfiledCodeHeapSize`） | C2 编译的完全优化代码，生命周期长 | 约 117.5MB |

换算：`240 − 5 = 235`MB 剩余，对半分得 `235 / 2 = 117.5`MB，即每段 `117.5 × 1024 = 120320`KB。实测佐证：某 JDK 构建的 `CodeHeap` 保留大小为 119168KB（约 116MB），与 117.5MB 的量级一致。

换算例（假设，非实测）：假设平均每个已编译方法（含方法头、重定位与元数据）占 2KB 机器码，则 non-profiled 段约能装 `120320 / 2 = 60160` 个 C2 方法，全堆约能装 `245760 / 2 = 122880` 个方法；若平均 1KB，则分别为约 120320 个与约 245760 个；若平均 4KB，则分别为约 30080 个与约 61440 个。只能这样估，不能当实测数：真实 nmethod 从几百字节到几十 KB 不等，一个方法还可能同时占 Tier 3、Tier 4 与 OSR 三个版本，适配器与桩代码另占 non-nmethods 段。

满了之后的行为见 §8：JIT 停止编译并打印 `CodeCache is full`，应用不崩、退回解释执行。`UseCodeCacheFlushing` 默认开启，清道夫会先把 `made zombie` 的旧编译产物扫出去腾地方；清不出来才会停编。只调大 `ReservedCodeCacheSize` 不查编译量，等于把问题推迟。

### 6.5 边界条件

- **阈值是计数器口径，不是时间口径**：方法 1 秒跑 200 次与 1 小时跑 200 次没有区别，攒够调用/回边次数才升层；调用频率低的方法永远到不了 C2。计数器还会随时间衰减（§5.1），偶发高频不会误判。
- **阈值是“检查点”，不是“精确触发点”**：解释器不是每次调用都查表，而是每满 128 次调用（`Tier0InvokeNotifyFreqLog = 7`，即 2⁷）或 1024 次回边（`Tier0BackedgeNotifyFreqLog = 10`，即 2¹⁰）才通知一次策略；Tier 3 代码每 1024 次调用、8192 次回边检查一次。所以名义阈值 200 次调用的方法，实际在第 256 次调用（128 × 2）时才被排队编译。
- **阈值会被编译队列长度放大**：6.1 谓词里的 `s` 由队列长度决定（`TierXLoadFeedback`），队列越长、有效阈值越高；编译高峰期方法升层更慢。调参时看到的阈值不是常量。
- **Tier4 阈值的版本差异不在数值、在实现**：Tier4 的 5000 / 600 / 15000 / 40000 从 JDK 8 到 JDK 21 没变过；变的是策略实现——JDK 8 用 `AdvancedThresholdPolicy`（`CompilationPolicyChoice = 3`），JDK 10 起 JDK-8202711 把两个策略合并为 `SimpleThresholdPolicy`（`CompilationPolicyChoice = 2`）。配套规则也在变：`MaxInlineLevel` 在 JDK 8 是 9，2020 年后的 OpenJDK 源码 `c2_globals.hpp` 里是 15；`MinInliningThreshold = 250` 在 JDK 17 及更早版本存在，JDK 21 已取消。抄数字先对版本，以目标 JDK 的 `PrintFlagsFinal` 为准。
- **内联预算可调，但调大不等于更好**：`-XX:MaxInlineSize`、`FreqInlineSize`、`InlineSmallCode` 都能改，但内联后方法体受 `DesiredMethodLimit = 8000` 与 IR 节点数上限（`NodeCountInliningCutoff`、`MaxNodeLimit`）约束；过度内联会让编译变慢、编译失败回退、CodeCache 膨胀。调之前先看 `PrintInlining` 的真实判定（"too big"、"hot method too big"、"inlining too deep"），不要凭感觉加数字。
- **CodeCache 满了不是 OOM**：应用不崩、只是 JIT 停编、退回解释执行（§8）；`UseCodeCacheFlushing` 默认开着，先清 `made zombie` 再停编。动态代理、脚本引擎大量生成类时，调大只是推迟打满，根因在编译量。
- **字节码超 8000 的方法根本不编译**：`DontCompileHugeMethods` / `HugeMethodLimit` 默认 8000 字节码，比内联预算更硬——这种方法全程解释执行，拆方法是唯一解法；64KB（65535 字节）则是 class 文件的方法体上限，编译直接报错，见小技巧节。

### 来源

- OpenJDK 源码 `src/hotspot/share/runtime/globals.hpp`（Tier3/Tier4 阈值、`CompileThreshold`、`OnStackReplacePercentage` 默认值），核验日期 2026-11-08。
- OpenJDK 源码 `src/hotspot/share/compiler/tieredThresholdPolicy.hpp`（升层判定谓词与 OSR 谓词），核验日期 2026-11-08。
- OpenJDK 源码 `src/hotspot/share/opto/c2_globals.hpp`（`MaxInlineSize = 35`、`MaxTrivialSize = 6`、`MaxRecursiveInlineLevel = 1`、`DesiredMethodLimit = 8000`；`MaxInlineLevel = 15` 为 2020 年源码值），核验日期 2026-11-08。
- OpenJDK 源码 `src/hotspot/share/runtime/tieredThresholdPolicy.cpp` 与 `simpleThresholdPolicy.cpp`（`InlineSmallCode`：x86 取 2000，SPARC / AArch64 取 2500；JDK-8202711 策略合并），核验日期 2026-11-08。
- 微软 OpenJDK 团队《How Tiered Compilation works in OpenJDK》（阈值公式与数值），核验日期 2026-11-08。
- OpenJDK Wiki《Server Compiler Inlining Messages》（内联判定语义：too big / hot method too big / inlining too deep / DesiredMethodLimit），核验日期 2026-11-08。
- JEP 197 Segmented Code Cache（JDK 9 起三段划分）与 OpenJDK 源码 `src/hotspot/share/code/codeCache.cpp`（剩余空间对半分）及 `codeCache.hpp`（分段默认开启条件），核验日期 2026-11-08。
- Oracle《Codecache Tuning》（JDK 8 文档，`ReservedCodeCacheSize` 默认 32M / 48M），核验日期 2026-11-08。
- OpenJDK jdk-21-ga 源码 `src/hotspot/share/opto/c2_globals.hpp`（`MaxInlineLevel = 15`；全文无 `MinInliningThreshold`，即 JDK 21 已取消该限制），核验日期 2026-11-08。

## 7. 实战应用

- **应用预热**：服务刚启动时解释执行 + 逐步编译，首批请求延迟显著高于稳态。压测和性能结论必须在预热后取；SLA 敏感的服务可在发布后先跑一轮核心链路自调用预热，再放真实流量。
- **基准测试必须用 JMH**：手写 `main` 计时会被 JIT 干扰得毫无意义——死代码消除（结果没人用，整段计算被删）、常量折叠、预热不足都会给出假数。JMH 的预热轮次 + `Blackhole` 消费结果就是为绕开这些坑设计的。
- **观察 JIT 行为**：`-XX:+PrintCompilation` 打印每次编译事件；`-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` 看内联决策；`-XX:+LogCompilation` 输出日志配合 JITWatch 图形化分析。
- **方法体大小**：超大方法（几千行）难以内联也难以整体优化，拆分首先是可读性问题，其次可能让 JIT 更好处理——但先测量再动手。

## 8. 疑难杂症

- **CodeCache 满，JIT 停摆**：编译产物放在 CodeCache（非堆独立区域），打满后 JIT 停止编译并打印 `CodeCache is full`，应用不崩但退回解释执行、性能塌方。处理：调大 `-XX:ReservedCodeCacheSize`，并查编译量为何异常（动态代理 / 脚本引擎大量生成类是常见源头），与 [JVM 故障排查](开发语言/Java/JVM/JVM-故障排查.md) 一起看。
- **跑一会儿变慢**：优先怀疑去优化抖动，或容器 CPU limit 让编译线程排不上队；先开 `PrintCompilation` 看编译/失效事件是否异常密集。

## 9. 常见误区

- `javac` 会做深度优化：实际它基本只做语法层面的事，性能优化的主战场在 JIT。
- 加 `final` 就能保证内联：是否内联由热点与体积决定；private / static 天然不可重写，无需靠 final。
- JIT 结果会持久化：默认不会，重启重新预热（CDS 只缓存类数据）；要启动即峰值看 GraalVM Native Image 等 AOT 方案，取舍完全不同。
- 逃逸分析 = 局部对象都不进堆：只有优化命中才不分配，字段过多、对象传进未内联方法都会让分析失败，别当确定性承诺。

## 原理剖析

本节只把原文已有章节给出的结论拆开讲清机制：不引入新概念，不补新数字，数字一律复用 §6 已核验的定量算例；凡原文没展开的点，文中明确标注，不作定论。

### 字节码的栈式执行：为什么先解释、再编译

原文 §1 给出的基本形态是「解释 + 编译」混合执行：Java 源码经 `javac` 编译为平台无关的字节码，JVM 先解释执行，运行期再把热点代码由 JIT 编译成本地机器码。`javac` 几乎不做优化，只有常量折叠等少数动作，真正的优化全在 JIT，且基于运行期的真实调用数据。

机制在 §2：每个方法有一个操作数栈和一张局部变量表，指令大多只操作栈顶，没有寄存器概念。原文 §4 的 `add` 例子就是这套模型的最小演示：`iload_1` 把局部变量 1（`a`，槽 0 是 `this`）压栈，`iload_2` 把 `b` 压栈，`iadd` 弹两个数相加再压回，`ireturn` 返回栈顶 `int`。这种栈式模型换来可移植性，代价是每次运算都要经过压栈、弹栈的搬运，这正是 §2 末尾说的「JIT 要消除的开销」。

所以「先解释」不是设计失误，而是分工：解释执行启动即用（§5.2 的 Tier 0），让代码先跑起来、同时收集真实的调用数据；JIT 再拿这些数据做 `javac` 做不了的优化。原文未展开：解释器逐条取指、分派指令的具体开销构成，原文未展开，不作定论。

### 热点探测：两个计数器、检查点与 OSR

原文 §5.1 的热点探测只用两个计数器：方法调用计数器与回边计数器（循环回跳时累加），两者之和达到阈值即判定热点、触发编译；计数随时间衰减，避免偶发高频误判。编译由后台线程异步进行，不阻塞业务线程，完成前继续解释执行。

§6.1 把这个判定写成谓词（`s` 是编译队列长度带来的放大系数，队列空时 `s = 1`）：`i > TierXInvocationThreshold × s`，或（`i > TierXMinInvocationThreshold × s` 且 `i + b > TierXCompileThreshold × s`）。Tier 3 的三项阈值是 200、100、2000，Tier 4 是 5000、600、15000。§6.1 的手算例可以反过来读懂这套机制：某方法每次调用跑 20 次循环回边，调用 150 次时 `i = 150`、`b = 150 × 20 = 3000`，`i + b = 3150`；`100 < 150 < 200`，光靠调用次数没到 200，但 `150 > 100 且 3150 > 2000`，混合口径成立，升到 Tier 3。升到 Tier 3 后继续跑，在 Tier 3 代码里累计 `i' = 700` 次调用、`b' = 15000` 次回边，`700 > 600 且 700 + 15000 = 15700 > 15000`，升到 Tier 4。没有循环的方法则要攒满 200 次调用才升 Tier 3、攒满 5000 次 Tier 3 调用才升 Tier 4。

还有两处机制决定了「阈值不是精确触发点」。一是 §6.5 的检查点：解释器不是每次调用都查表，而是每满 128 次调用（`Tier0InvokeNotifyFreqLog = 7`，即 2⁷）或 1024 次回边（`Tier0BackedgeNotifyFreqLog = 10`，即 2¹⁰）才通知一次策略；Tier 3 代码每 1024 次调用、8192 次回边检查一次。所以名义阈值 200 次调用的方法，实际在第 256 次调用（128 × 2）时才被排队编译。二是 §6.5 的队列放大：谓词里的 `s` 由队列长度决定，队列越长、有效阈值越高，编译高峰期方法升层更慢。

只被调用一次、但内部循环跑很久的方法等不到「下一次调用」，走 §5.1 的 OSR（栈上替换）：方法还在栈上就把当前帧换成编译版本。§6.2 的口径只有回边次数 `b`：分层编译下 OSR 到 Tier 3 要 `b > 60000`（`Tier3BackEdgeThreshold`），再到 Tier 4 要在 Tier 3 代码里回边再满 40000 次（`Tier4BackEdgeThreshold`）；关掉分层编译时 OSR 阈值是算出来的，`CompileThreshold × OnStackReplacePercentage / 100 = 10000 × 140 / 100 = 14000`，分层开启时这两个参数不参与 OSR 判定。

原文未展开：计数器随时间衰减的具体周期与幅度，原文未展开，不作定论；放大系数 `s` 由队列长度算出的具体公式，原文未展开，不作定论。

### 分层编译与内联：C1 剖析给 C2 用，内联按两套尺子量

原文 §5.2 的分层是：Tier 0 解释执行，Tier 1–3 为 C1 编译（编译快、优化浅，不同层带不同程度的性能剖析），Tier 4 为 C2 编译（编译慢、优化深，长期热点才值得）。典型路径是解释 → C1（带剖析）→ 收集调用频率、分支走向、接收者类型 → C2 激进优化。C1 的作用不只是「先快一点」，更是给 C2 攒剖析数据；C2 的激进优化（§5.3）之所以敢做投机假设，正因为有这份运行期数据打底。

内联是 §5.3 的核心优化：被调用方法体嵌入调用处，消除调用开销并打开后续优化空间；虚方法要先去虚化。去虚化的机制是：剖析显示接收者只有一个实现类时按单态直接内联，插类型守卫，猜错就去优化。逃逸分析则是下面几项的前提：分析对象引用是否逃出方法 / 线程，命中后才有标量替换（不逃逸对象拆成标量放栈上 / 寄存器，对象根本不分配、随栈帧消失）与同步消除（对不逃逸对象上的 `synchronized` 直接去锁，如局部 `StringBuffer`）。

§6.3 把内联预算拆成两套尺子：一套量被调方法的字节码大小，一套量它已编译成的机器码大小。字节码尺子：`MaxInlineSize = 35`（冷调用点，超过 35 字节就不内联）、`FreqInlineSize = 325`（热调用点，x86_64）、`MaxTrivialSize = 6`（≤ 6 字节的极小方法无条件优先内联）、`DesiredMethodLimit = 8000`（内联后调用方方法体超过 8000 字节码就停止继续内联）。机器码尺子：`InlineSmallCode = 2000`（x86）/ 2500（SPARC / AArch64），被调方法已有编译版本且机器码小于该值时才内联，大于该值判 "already compiled into a big method"，中等的按 `InlineSmallCode / 4`（x86 即 500）判。层数上：`MaxInlineLevel = 9`（JDK 8）；15（2020 年后源码），`MaxRecursiveInlineLevel = 1`；JDK 17 还要求被调方法至少跑过 `MinInliningThreshold = 250` 次才考虑内联，JDK 21 已取消这条限制。

§6.3 的手算例把两套尺子的关系讲清了：某方法字节码 40 字节，冷调用时 `40 > 35`，超过 `MaxInlineSize`，判 "too big"，不内联——只差 5 字节也进不去。跑热后调用点变热，改按 `FreqInlineSize` 算：`40 ≤ 325`，还剩 `325 − 40 = 285` 字节余量，可以内联；热门槛是冷门槛的 `325 / 35 ≈ 9.29` 倍。但「可以」不等于「一定」：还得同时满足内联深度 ≤ `MaxInlineLevel`、内联后方法体 ≤ `DesiredMethodLimit = 8000` 字节码、JDK 17 下被调方法已跑满 `MinInliningThreshold = 250` 次。超过预算的调用点不内联，但方法本身照常被 JIT 编译——丢的只是跨方法边界的优化，不是编译本身。

原文未展开：C1 收集的剖析数据的具体结构与保存方式，原文未展开，不作定论；类型守卫失效的判定细节与去优化后重新剖析的具体步骤，原文未展开，不作定论；逃逸分析本身的算法实现，原文未展开，不作定论。

### CodeCache 与去优化：编译产物放哪、投机失败怎么办

编译产物存在 CodeCache（非堆独立区域，见 §8）。§6.4 的默认大小分两种情况：分层编译开启（默认，JDK 8 起）为 240MB（即 48MB × 5，`240 × 1024 × 1024 = 251658240` 字节 = 245760KB）；分层编译关闭为 48MB（Server VM；Client VM 为 32MB）。JDK 9 起（JEP 197）CodeCache 默认分段成三个堆，按「非方法段先取固定值、剩余两段对半分」算出：non-nmethods 约 5MB，profiled nmethods 约 117.5MB，non-profiled nmethods 约 117.5MB；换算为 `240 − 5 = 235`MB 剩余，对半分得 `235 / 2 = 117.5`MB，即每段 `117.5 × 1024 = 120320`KB。

满了之后的行为见 §8 与 §6.5：JIT 停止编译并打印 `CodeCache is full`，应用不崩、退回解释执行。`UseCodeCacheFlushing` 默认开启，清道夫会先把 `made zombie` 的旧编译产物扫出去腾地方；清不出来才会停编。只调大 `ReservedCodeCacheSize` 不查编译量，等于把问题推迟。

去优化（§5.3）是这套投机优化的另一面：投机假设被打破时（如新加载了子类实现、分支走向突变），已编译代码作废、退回解释执行重新剖析。偶发正常，高频会造成性能抖动，看 `PrintCompilation` 里是否反复出现 `made not entrant`。

原文未展开：清道夫触发清理的具体条件与扫描范围，原文未展开，不作定论；一个方法同时占 Tier 3、Tier 4 与 OSR 三个版本时各版本在 CodeCache 里的去留规则，原文未展开，不作定论。

## 实战案例

### 案例一：40 字节的方法冷调用时不内联，跑热后才内联

现象：某方法字节码 40 字节（§6.3 手算例的同一个量级），冷调用时没有被内联，跑热后才内联上；原文 §6.3 手算例指出这种方法的性能曲线会先平后陡。

排查：先拿 `javap -c` 看方法到底多大（§6.3：`javap -c` 看到的就是这个字节数，先量大小再谈为什么没内联），确认是 40 字节；再开 `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` 看内联决策（§7），冷调用时应看到 "too big" 的判定。对照 §6.3 的预算：冷调用按 `MaxInlineSize = 35` 算，`40 > 35`，超过 5 字节也进不去；跑热后调用点变热，改按 `FreqInlineSize = 325` 算，`40 ≤ 325`，还剩 `325 − 40 = 285` 字节余量，热门槛是冷门槛的 `325 / 35 ≈ 9.29` 倍。

结论：不是方法写错了，而是冷热两套预算不同。跑热后可以内联还不等于一定内联，还得同时满足内联深度 ≤ `MaxInlineLevel`（JDK 8 为 9；2020 年后源码为 15）、内联后方法体 ≤ `DesiredMethodLimit = 8000` 字节码、JDK 17 下被调方法已跑满 `MinInliningThreshold = 250` 次（JDK 21 已取消这条限制），以上数字均出自 §6.3；若仍不内联，先看 `PrintInlining` 的真实判定（"too big"、"hot method too big"、"inlining too deep"），不要凭感觉加数字（§6.5）。

### 案例二：只调用一次的长循环方法迟迟不编译

现象：一个只被调用一次、但内部循环跑很久的方法（如 `main` 里的长循环，§6.2 的原话），等不到「下一次调用」，迟迟没有走上编译版本。

排查：先确认是不是在等方法调用次数——§6.1 的升层口径看的是调用次数 `i` 与回边次数 `b` 之和，单次调用的方法靠调用次数攒不够 Tier 3 的 200 次、Tier 4 的 5000 次；再看回边计数：分层编译下 OSR 到 Tier 3 要 `b > 60000`（`Tier3BackEdgeThreshold`），再到 Tier 4 要在 Tier 3 代码里回边再满 40000 次（`Tier4BackEdgeThreshold`），以上数字均出自 §6.2。排查时看 `-XX:+PrintCompilation` 输出（§7）：行首带 `%` 的是 OSR 编译（循环里正在跑的方法被栈上替换），不带的是普通方法编译；回边没满 60000 次之前，看不到该方法的 OSR 编译事件是符合口径的。注意不要拿非分层模式的 14000 去套分层编译的 JVM：`10000 × 140 / 100 = 14000` 只在关掉分层编译（`-XX:-TieredCompilation`）时才用（§6.2）。

结论：这类方法不靠调用次数升层，靠回边计数走 OSR；回边满 60000 次才到 Tier 3、再满 40000 次才到 Tier 4。循环跑得不够久、回边没攒够之前继续解释执行是正常行为，不是 JIT 故障；若回边早已超过 60000 次仍无 OSR 编译事件，再按 §8 的思路查编译线程与 CodeCache 是否异常。

## 小技巧与冷知识

【小技巧】内联不是“方法越小越好”那么笼统，是有两条硬门槛：普通方法默认只内联字节码不超过 35 字节的（`-XX:MaxInlineSize=35`），热点方法放宽到 325 字节（`-XX:FreqInlineSize=325`）。一个 40 字节的方法冷调用时内联不上、跑热了才内联得上，性能曲线会先平后陡；排障时先拿 `javap -c` 看方法到底多大，再谈它为什么没被内联。

【小技巧】`PrintCompilation` 的输出会读才有价值：行首带 `%` 的是 OSR 编译（循环里正在跑的方法被栈上替换），不带的是普通方法编译；数字是编译序号和层级，`made not entrant` 是去优化后旧版本作废，之后清道夫再把它标成 `made zombie` 回收。同一方法反复出现“编译—made not entrant—再编译”，就是原文说的去优化抖动，先看这条再谈调参。

【小技巧】想看 JIT 到底生成了什么汇编，`-XX:+PrintAssembly` 光加参数不够：它需要 hsdis 反汇编插件，没装时只会打印一条找不到插件的警告、看不到任何汇编，别误以为参数没生效。装好 hsdis 后配合 `-XX:PrintAssemblyOptions=intel` 看 Intel 语法输出，定位“这段循环到底有没有被展开”比猜快得多。

【小技巧】`javap -v` 看常量池时顺带看 `ldc`：字符串字面量存在常量池里，字节码用 `ldc` 按索引取，同一个字面量在多个地方出现也只在常量池里存一份。循环里反复写同一个长字符串字面量不会每次都新建对象，`ldc` 拿到的就是常量池里那一份；真正每次新建的是 `new String(...)` 和拼接结果，排障时先分清“取常量”和“新建对象”。

【小技巧】Lambda 不捕获变量时每次调用拿到的是同一个实例：不捕获的 lambda 由 `invokedynamic` 的引导方法只生成一次、之后复用，捕获了变量的每次调用都会新建一个实例。热循环里写 `list.stream().filter(x -> x > 0)` 这种不捕获的 lambda，实例开销可以忽略；一旦捕获了外部变量，循环每次迭代都在分配，逃逸分析也救不了它。

【小技巧】JDK 9+ 的字符串拼接不是 `StringBuilder` 了：它走 `invokedynamic` + `StringConcatFactory`，默认策略是 `MH_INLINE_SIZED_EXACT`，拼接逻辑由方法句柄在运行期组合。想换策略可以加 `-Djava.lang.invoke.stringConcat=BC_SB` 切回字节码版 `StringBuilder` 做法做对照；怀疑拼接是瓶颈时，先确认你测的是哪个 JDK 版本的拼接实现，再谈优化。

【小技巧】怀疑标量替换没生效，别只盯着 GC 日志：逃逸分析默认开启（`-XX:+DoEscapeAnalysis`），想验证“这个对象到底分没分配”，用 `-XX:-DoEscapeAnalysis` 关掉它对照跑一轮——关掉后分配量和 GC 次数明显上升，说明原来确实在做标量替换；关掉后没变化，说明对象本来就逃逸了，优化根本没命中，问题在代码写法不在参数。

【小技巧】CodeCache 打满别只会调大：默认值 240MB，JDK 11 起还默认分段（SegmentedCodeCache）分成三块堆。满了之后先确认 `UseCodeCacheFlushing` 是开着的（默认 true），它会让清道夫把 `made zombie` 的旧编译产物扫出去腾地方；只调大不清理，动态生成类多的应用迟早还会打满，原文说的“查编译量为何异常”才是根。

【冷知识】class 文件开头的魔数 `0xCAFEBABE` 是 James Gosling 亲自挑的：Java 的名字就来自咖啡，`CAFEBABE` 是“咖啡馆美女”（cafe babe）的谐音梗，写进文件头当签名。之后所有 JVM 语言的 class 文件都沿用它，读文件头前 4 个字节就能认出这是不是一份合法的 class 文件。

【冷知识】单个方法的字节码有 64KB 上限：Code 属性的长度字段是 4 字节，但规范另有限制，方法体超过 65535 字节编译就报 `code too large`。自动生成代码（比如模板引擎、动态代理手写巨型方法）最容易撞这条线，解法不是调参数，而是把方法拆小——上限写在规范里，任何参数都绕不过去。

## 10. 面试与自测

- class 文件由哪几部分组成？常量池存什么？
- 五种方法调用指令分别用于什么场景？Lambda 为什么用 `invokedynamic`？
- 分层编译各层是什么？为什么不直接全用 C2？
- 逃逸分析能带来哪三项优化？标量替换和栈上分配什么关系？
- 什么是去优化？什么情况会触发？
- 为什么手写计时基准不准？JMH 解决了哪几个问题？
- 一个 40 字节的方法冷调用时为什么内联不上？它要满足什么条件才能在跑热后被内联，方法本身又要先经过哪两级编译？

### 参考答案

1. 依次为：魔数 `0xCAFEBABE`、版本号（JDK 8 主版本 52、JDK 17 为 61，高版本 class 不能跑低版本 JVM）、常量池、访问标志、类/父类/接口索引、字段表、方法表（字节码在方法的 Code 属性里）、属性表。常量池存放类名、方法名、字符串等字面量与符号引用，指令靠索引引用它。

2. `invokestatic` 用于静态方法，编译期绑定；`invokespecial` 用于构造器、private 方法和父类方法；`invokevirtual` 用于普通虚方法，按接收者实际类型经虚方法表分派；`invokeinterface` 用于接口方法，分派略慢于 virtual；`invokedynamic`（JDK 7）把调用逻辑延迟到运行期决定。Lambda 用 `invokedynamic`，因为 Lambda 的实现方式不需要在编译期写死为某个类的方法调用，而是运行期由引导方法决定如何生成/绑定实现。

3. Tier 0 为解释执行，启动即用但慢；Tier 1–3 为 C1 编译，编译快、优化浅、带不同程度的性能剖析；Tier 4 为 C2 编译，编译慢、优化深。典型路径是解释 → C1（带剖析）→ 收集调用频率、分支走向、接收者类型 → C2 激进优化。不直接全用 C2，是因为 C2 编译成本高，会拖慢启动，且短生命周期代码不值得深优化；JDK 8 起分层编译默认开启，不要手动关掉。

4. 逃逸分析是下面三项的前提：标量替换、同步消除，以及与之并列的循环优化收益（正文把标量替换、同步消除列为逃逸分析直接带来的两项，循环优化亦在同一优化组内）。标量替换与栈上分配的关系：标量替换把不逃逸对象拆成标量放栈上或寄存器，对象根本不分配、随栈帧消失，收益最大；它比“整个对象放栈上”更彻底，是栈上分配的一种实现效果，但只有分析命中时才成立，不是确定性承诺。

5. 去优化是指投机假设被打破时，已编译代码作废、退回解释执行重新剖析。触发情形如：新加载了子类实现（打破单态去虚化假设）、分支走向突变。偶发正常，高频会造成性能抖动，排查看 `PrintCompilation` 里是否反复出现 `made not entrant`。

6. 手写计时不准，因为会被 JIT 干扰：死代码消除（结果没人用，整段计算被删）、常量折叠、预热不足都会给出假数。JMH 针对性解决这些问题：用预热轮次让 JIT 达到稳态，再用 `Blackhole` 消费计算结果防止死代码消除，从而得到可信的基准。
7. 40 > MaxInlineSize = 35，冷调用判 "too big" 不内联；跑热后调用点变热，40 ≤ FreqInlineSize = 325 可以内联，还需内联深度 ≤ MaxInlineLevel、内联后方法体 ≤ DesiredMethodLimit = 8000 字节码（JDK 17 下被调方法还须跑满 MinInliningThreshold = 250 次）；方法本身先在调用 200 次（或调用 100 次以上且调用 + 回边 > 2000）时升到 Tier 3（C1 带剖析），再在 Tier 3 下调用 5000 次（或 600 次以上且调用 + 回边 > 15000）时升到 Tier 4（C2）。

## 11. 关联知识

### 前置知识

- [JVM 类加载](开发语言/Java/JVM/JVM-类加载.md)：字节码从哪来、验证阶段校验什么，与 class 结构对应（引用原文 §2、§11）。
- [JVM 内存模型](开发语言/Java/JVM/JVM-内存模型.md)：运行时数据区与 CodeCache 的位置（引用原文 §6.4、§8、§11）。
- [JVM GC 调优](开发语言/Java/JVM/JVM-GC调优.md)：逃逸分析与标量替换如何减轻 GC 压力（引用原文 §5.3、§11）。
- [JVM 故障排查](开发语言/Java/JVM/JVM-故障排查.md)：CodeCache 满、性能塌方的定位流程（引用原文 §8、§11）。

- [JVM 类加载](开发语言/Java/JVM/JVM-类加载.md)：字节码从哪来、验证阶段校验什么，与 class 结构对应。
- [JVM 内存模型](开发语言/Java/JVM/JVM-内存模型.md)：运行时数据区与 CodeCache 的位置。
- [JVM GC 调优](开发语言/Java/JVM/JVM-GC调优.md)：逃逸分析与标量替换如何减轻 GC 压力。
- [JVM 故障排查](开发语言/Java/JVM/JVM-故障排查.md)：CodeCache 满、性能塌方的定位流程。
