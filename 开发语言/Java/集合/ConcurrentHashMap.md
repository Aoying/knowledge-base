---
知识点: ConcurrentHashMap
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-11-09
关联:
  - HashMap
  - Java-并发编程
  - Java
标签:
  - Java
  - 并发
  - 集合
---

# ConcurrentHashMap

## 1. 核心概念

- 定位：`HashMap` 的线程安全版，`java.util.concurrent` 包下专为高并发读写设计。
- 目标：读基本无锁、写只锁局部，吞吐量远高于全表锁方案。
- 不允许 `null` key、不允许 `null` value，放入直接抛 `NullPointerException`。

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put(null, "v");   // NullPointerException
map.put("k", null);   // NullPointerException
```

- 为什么禁 `null`：并发下 `get(k)` 返回 `null` 无法区分两种情况：key 不存在，还是 key 存在但 value 就是 `null`。
- 单线程 `HashMap` 可用 `containsKey(k)` 再判断。并发下两次调用之间别的线程可能插入或删除该 key，`containsKey` 配合也不可靠，无法消除歧义。
- 取舍：宁可禁掉 `null`，也不留一个并发下语义不确定的口子。

## 2. 基础知识

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.get("a");                    // 读，无锁
map.putIfAbsent("a", 2);         // 不存在才放入，原子
map.remove("a", 1);              // 值等于 1 才删除，原子
map.replace("a", 1, 3);          // 值等于 1 才替换，原子
map.compute("a", (k, v) -> v == null ? 1 : v + 1);
map.merge("a", 1, Integer::sum);
// 初始容量向上取 2 的幂；JDK 8 的 concurrencyLevel 仅作容量估算，不再决定锁数量
Map<String, String> m2 = new ConcurrentHashMap<>(64, 0.75f, 16);
```

- 单个方法原子，不等于复合操作原子：先 `get` 判断、再 `put` 不是原子的，必须用 `compute` 系方法，见第 6 节。
- 三种线程安全 Map 对比：

| 维度 | Hashtable | Collections.synchronizedMap | ConcurrentHashMap |
|---|---|---|---|
| 锁粒度 | 全表 synchronized 方法 | 全表一把互斥锁包装所有方法 | JDK7 分段锁 / JDK8 锁单个桶头 |
| 性能 | 低，并发退化为串行 | 低，与 Hashtable 相近 | 高，读写可并行 |
| null 支持 | 不允许 null key/value | 取决于被包装 Map，HashMap 允许 | 不允许 null key/value |
| 迭代器 | 弱一致性（Enumeration/迭代器） | 快速失败，迭代需手动同步全表 | 弱一致性，不抛 ConcurrentModificationException |
| 复合操作 | 无原子复合 API | 无，需外部加锁 | putIfAbsent/compute/merge 原子 |

## 3. 进阶知识

### 3.1 弱一致性迭代器

- 迭代器不抛 `ConcurrentModificationException`，与 `HashMap` 的快速失败不同。
- 迭代中允许其他线程增删：看到的是某一时刻的快照式视图，已遍历的不会回退，未遍历的可能被改，可能漏掉新插入的，也可能看到刚被删的。

```java
for (Map.Entry<String, Integer> e : map.entrySet()) {
    // 其他线程此时 put/remove 不会让这里抛异常，结果容忍少量误差即可
    System.out.println(e.getKey() + "=" + e.getValue());
}
```

- 适用：遍历统计、监控打印、缓存清理。不适用：要求数据冻结的强一致对账，需外部加锁或换方案。

### 3.2 视图与并行操作

- `keySet()`、`values()`、`entrySet()` 视图支持并发修改；迭代器不支持 `remove()`，直接调 `map.remove(key)`。
- `Set<String> set = ConcurrentHashMap.newKeySet();` 可直接当并发 Set 用。
- JDK 8 的 `forEach/search/reduce(parallelismThreshold, ...)` 支持并行处理，阈值设 `Long.MAX_VALUE` 即串行。

## 4. 原理剖析

### 4.1 JDK 7：Segment 分段锁

- 结构：`Segment[]` + 每段内部一个 `HashEntry[]`，每段是一个继承 `ReentrantLock` 的小 HashMap。
- 默认 16 段，并发度 = 段数 = 16，理想情况下最多 16 个线程同时写不同段。
- 定位：先按 hash 高位选 Segment，再按段内 hash 选桶；`get` 走 volatile 读基本无锁，`put` 只锁目标段。

```text
ConcurrentHashMap
 ├─ Segment[0] (ReentrantLock) -> HashEntry[] -> 链表
 ├─ Segment[1] (ReentrantLock) -> HashEntry[] -> 链表
 └─ ... Segment[15]
```

- 缺点：并发度被段数定死，扩容只扩单段，段数本身不扩。

### 4.2 JDK 8：数组 + 链表 + 红黑树，锁细化到桶

- 去掉 `Segment`，结构回归与 `HashMap` 一致：`Node[] table`；链表长度 >= 8 且数组长度 >= 64 转红黑树，节点 <= 6 退化回链表。
- `Node` 的 `val` 和 `next` 都是 `volatile`，保证写后其他线程立刻可见：

```java
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K, V> next;
}
```

- 写入流程（`putVal` 伪代码）：

```java
if (tab == null) initTable();
else if ((f = tabAt(tab, i)) == null) {
    if (casTabAt(tab, i, null, new Node<>(hash, key, val))) break; // 空桶：CAS 无锁插入
} else if (f.hash == MOVED) {
    tab = helpTransfer(tab, f); // 正在扩容：先协助扩容
} else {
    synchronized (f) { /* 冲突：只锁桶头节点，遍历链表/红黑树插入或更新 */ }
}
```

- 相比 JDK 7：锁粒度从「一段」细到「一桶」，冲突概率大幅降低；用 `synchronized` 是因桶头锁生命周期短、JVM 对其优化足够，且省内存。

### 4.3 扩容协助：多线程一起搬

- 触发：元素数超过 `容量 * 0.75`，或链表转树时容量不足。
- 已迁移的桶放 `ForwardingNode`（hash = `MOVED = -1`）；其他线程读写命中它时不等待，调用 `helpTransfer` 一起搬。
- `transfer` 按步长分工，默认每线程至少负责 16 个桶，避免抢同一段活。
- `sizeCtl` 总控字段：

| sizeCtl 取值 | 含义 |
|---|---|
| `0` | 未初始化，用默认容量 |
| `> 0` | 未初始化时是初始容量；已初始化时是下次扩容阈值 |
| `-1` | 正在初始化 |
| `< -1` | 正在扩容，低 16 位编码参与扩容的线程数 |

### 4.4 计数：baseCount + CounterCell 分散计数

- 不用一把锁维护全局计数器，否则 `put` 全卡在计数上。做法类 `LongAdder` 思想：无冲突先 CAS `baseCount`，冲突则落到 `CounterCell[]` 分散累加。
- `size()` = `baseCount` + 所有 `CounterCell` 之和；各分片不在同一时刻统计，所以 `size()` 是估算值、并发下不精确。`mappingCount()` 返回 `long`，同样是近似值。

## 5. 实战应用

### 5.1 computeIfAbsent 做缓存 / 单例初始化

```java
ConcurrentHashMap<String, ExpensiveObject> cache = new ConcurrentHashMap<>();
ExpensiveObject obj = cache.computeIfAbsent("key", k -> createExpensive(k));
```

- 同一 key 的映射函数只执行一次，其余线程等结果，不重复创建。
- 映射函数不能太慢：执行期间占用该桶，慢函数会拖住同桶其他 key；不能在函数里递归回写本 map，会死锁，见 6.1。

### 5.2 merge / LongAdder 式计数统计词频

```java
ConcurrentHashMap<String, Long> freq = new ConcurrentHashMap<>();
for (String word : words) freq.merge(word, 1L, Long::sum); // 不存在放 1，存在累加，原子
// 更高并发：ConcurrentHashMap<String, LongAdder>，computeIfAbsent 后 increment()
```

### 5.3 putIfAbsent 抢占式占位

```java
ConcurrentHashMap<String, Task> running = new ConcurrentHashMap<>();
if (running.putIfAbsent(taskId, task) == null) { // 返回 null = 抢到了
    task.run();
    running.remove(taskId);
} // 返回非 null = 别人已在跑，跳过，避免重复执行
```

## 6. 疑难杂症

### 6.1 computeIfAbsent 递归调用自身：死锁 / IllegalStateException

```java
map.computeIfAbsent("a", k -> {
    map.put("b", 1);          // 映射函数里改同一个 map
    return map.computeIfAbsent("a", k2 -> 1); // 递归回写自身 key
});
```

- 映射函数执行时持有桶锁，递归回写可能等同一把锁直接卡死，某些路径抛 `IllegalStateException: Recursive update`。
- 解法：映射函数只做纯计算，依赖数据提前在锁外算好再传入。

### 6.2 用 size() == 0 判断后初始化：check-then-act 竞态

```java
if (map.size() == 0) initDefaults(map); // A、B 两线程同时判 0，都执行初始化，互相覆盖
```

- `size()` 本身是估算，且判断与执行之间无原子性。解法：逐个 `putIfAbsent` 占位，或用独立 `AtomicBoolean` 控制只初始化一次。

### 6.3 复合操作 get + put 非原子

```java
Integer v = map.get("count");
map.put("count", v == null ? 1 : v + 1); // 两线程读到同一旧值，计数少加
```

- 解法：改用 `merge(key, 1, Integer::sum)` 或 `compute`，一步完成读-改-写。

## 7. 常见误区

- 以为读全程无锁就等于强一致：无锁只保证可见性和不阻塞，读到的可能是写入前的值，`size()`、`containsValue` 都是弱一致结果。
- 以为 `synchronizedMap` 性能差不多：它是全表一把锁，高并发下吞吐差距是数量级的。
- 拿它当强一致注册表用：它没有跨多个 key 的事务性，服务注册、分布式协调应用 ZooKeeper / etcd，不要用单机 map 冒充。
- 以为 map 线程安全，value 就不用管：value 对象被多线程改时，仍需自身同步或做成不可变对象。

## 8. 性能优化

- 预估容量初始化，避免频繁扩容：容量按 `expectedSize / 0.75 + 1` 给。
- key 的 `hashCode` 要散列均匀，烂 hash 会让同桶冲突和锁竞争集中，扰动函数只能救一部分。
- 高频纯计数优先 `LongAdder` 分片计数后汇总，不要硬挤一个 key 反复 `merge`。
- `compute` 系方法的函数保持短小，慢 IO 先在锁外算好；读多写极少可考虑不可变快照 + `volatile` 引用，不必默认上它。

## 9. 版本差异

| 维度 | JDK 7 | JDK 8+ |
|---|---|---|
| 结构 | Segment + HashEntry 链表 | Node 数组 + 链表 + 红黑树 |
| 锁 | ReentrantLock 分段锁，默认 16 段 | CAS + synchronized 锁桶头节点 |
| 并发度 | 固定为段数 | 理论上为桶数，随容量增长 |
| 扩容 | 单段扩容 | 全表扩容，多线程协助 transfer |
| 计数 | 分段统计后汇总 | baseCount + CounterCell 分散计数 |
| 新增 API | 无 | compute / merge / forEach / search / newKeySet |

## 10. ConcurrentHashMap 容量与并发度定量：表大小、树化与扩容协助算一算

本节只算数，不重复第 1–9 节的定性结论：默认初始容量、负载因子、树化阈值、扩容协助步长、CounterCell 分散计数的机制见第 2、4 节；下面把它们换算成表大小、阈值、概率与耗时。

### 10.1 初始容量与阈值：从 16、0.75 到 12

- 默认值（OpenJDK 源码）：`DEFAULT_CAPACITY = 16`，`LOAD_FACTOR = 0.75f`。扩容阈值 = 容量 × 负载因子，默认表阈值 = 16 × 0.75 = 12，源码写作 `n - (n >>> 2)`，即 16 − 4 = 12。
- 阈值的含义：元素数超过 12（即放入第 13 个元素）时触发扩容；阈值不是“最多能装多少”，而是“装到多少就翻倍”。
- 初始容量的取法分两种构造器写法，算式不同、结果一致的方向相同：
  - 单参构造器 `new ConcurrentHashMap<>(N)`：`sizeCtl = tableSizeFor(N + (N >>> 1) + 1)`，即先算 1.5N + 1，再向上取 2 的幂；
  - 三参构造器 `new ConcurrentHashMap<>(N, loadFactor, concurrencyLevel)`：按 `1 + N / loadFactor` 再 `tableSizeFor` 向上取 2 的幂。
  - 两种写法都是“按负载因子反推所需桶数，再取整到 2 的幂”，不要把传入的 N 直接当成表大小。

| 预期元素数 N | 1.5N + 1 | 初始表大小（向上取 2 的幂） | 扩容阈值（表大小 × 0.75） |
| --- | --- | --- | --- |
| 100 | 151 | 256 | 192 |
| 1000 | 1501 | 2048 | 1536 |
| 10000 | 15001 | 16384 | 12288 |

- 向上取整的代价：N = 100 时表是 256 个桶，只装 100 个元素，利用率 100 ÷ 256 ≈ 39%，阈值 192 仍留近一倍余量——多占的内存换来的是扩容次数的减少，见 10.2。

### 10.2 容量估小了：从 16 起翻倍要扩几次、累计搬多少桶

- 每次扩容表大小翻倍，阈值同步翻倍：16 → 32 → 64 → 128 → 256 → 512 → 1024 → 2048 → 4096 → 8192 → 16384，对应阈值 12 → 24 → 48 → 96 → 192 → 384 → 768 → 1536 → 3072 → 6144 → 12288。

| 预期元素数 N | 扩容次数 | 终表大小 | 累计搬移桶数 |
| --- | --- | --- | --- |
| 100 | 4 | 256 | 16 + 32 + 64 + 128 = 240 |
| 1000 | 7 | 2048 | 16 + 32 + 64 + 128 + 256 + 512 + 1024 = 2032 |
| 10000 | 10 | 16384 | 16 + 32 + … + 8192 = 16368 |

- 累计搬移量的速算：每次扩容都要把旧表全部搬一遍，累计搬移 = 各次旧表大小之和 = 终表大小 − 16。验算：256 − 16 = 240；2048 − 16 = 2032；16384 − 16 = 16368。
- 结论：N = 10000 时从默认 16 起步要扩容 10 次、累计搬 16368 个桶；直接 `new ConcurrentHashMap<>(10000)` 一次建成 16384 的表，搬移量为 0。容量估小付出的不是一次扩容，而是一串翻倍扩容的累计搬移。

### 10.3 树化阈值：链长 ≥ 8 的概率与查找代价

- 三个常量（OpenJDK 源码）：`TREEIFY_THRESHOLD = 8`，`UNTREEIFY_THRESHOLD = 6`，`MIN_TREEIFY_CAPACITY = 64`。8 与 6 之间留 2 的迟滞带，避免链长在 7、8 之间抖动时反复树化、退树。
- 先扩容、后树化：表大小 < 64 时，某桶链长到 8 触发的是扩容而不是树化；只有表 ≥ 64，链长 ≥ 8 的桶才会转红黑树。表 16、某桶链长 8 时，表先扩到 32，链被拆分，而不是直接树化。
- 单桶链长 ≥ 8 的概率（泊松近似）：假设哈希均匀、负载因子 0.75，单桶元素数近似服从 Poisson(λ = 0.75)，则 P(X ≥ 8) = 1 − Σ(k = 0..7) e^(−0.75) × 0.75^k / k! ≈ 1.28 × 10^(−6)，约每 782000 个桶才出现一个。
  - 表 1024 时，这样的桶期望个数 = 1024 × 1.28 × 10^(−6) ≈ 0.0013 个；
  - 表 4096 时，期望个数 = 4096 × 1.28 × 10^(−6) ≈ 0.0052 个，约每 192 张 4096 的表才有一张含一个链长 ≥ 8 的桶。
  - 结论：均匀哈希下树化是极小概率的兜底；真实场景里出现长链，多半是哈希函数不好或 key 的 hashCode 扎堆，而不是“数据量大”的必然结果。
- 查找代价对比：链长 n 时链表最坏比较 n 次；红黑树深度约 log₂n，最坏高度 ≤ 2 × log₂(n + 1)。

| 链长 n | 链表最坏比较次数 | 红黑树深度约 log₂n | 红黑树最坏高度 |
| --- | --- | --- | --- |
| 8 | 8 | 3 | 7 |
| 16 | 16 | 4 | 9 |
| 64 | 64 | 6 | 13 |

- 链长 64 时，链表最坏 64 次比较，红黑树约 6 次（最坏 13 次），差 5–10 倍；链长 8 时只差 8 次对 3 次，树化收益有限。从比较次数看，这与阈值取 8 而非更小值的取舍方向一致（个人推断，非官方注释原话）。

### 10.4 扩容协助：步长、分工与耗时

- 步长口径（OpenJDK JDK 8 `transfer` 源码，已核实）：`if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE) stride = MIN_TRANSFER_STRIDE;`，其中 `MIN_TRANSFER_STRIDE = 16`，n 为旧表大小，NCPU 为可用处理器数。(n >>> 3) / NCPU 即 n ÷ 8 ÷ NCPU，与“表大小 ÷（核数 × 8）”是同一算式（整数除法）。
- 步长与分工换算：

| 旧表大小 n | NCPU | (n >>> 3) / NCPU | 步长 | 分块数 n / 步长 |
| --- | --- | --- | --- | --- |
| 1024 | 8 | 128 / 8 = 16 | 16 | 64 |
| 1024 | 16 | 128 / 16 = 8 → 取下限 16 | 16 | 64 |
| 4096 | 8 | 512 / 8 = 64 | 64 | 64 |
| 4096 | 16 | 512 / 16 = 32 | 32 | 128 |

- 规律：核数翻倍不一定让步长减半——1024 的表在 16 核下仍被下限 16 钳住，分块数不变；只有表足够大时，步长才随核数缩小。
- 耗时对比（假设：每桶搬移耗时相同、忽略线程争用与调度开销；理想情况，非官方承诺）：总搬移量恒为 n 个桶，协助搬表省的是时间，不是工作量。
  - 表 1024：单线程搬需 1024t；4 线程协助，每线程分 64 ÷ 4 = 16 块 × 16 桶 = 256t，理想加速 4 倍；
  - 表 4096（NCPU = 8，步长 64）：单线程搬需 4096t；4 线程协助，每线程分 64 ÷ 4 = 16 块 × 64 桶 = 1024t，理想加速同样是 4 倍。
  - 线程数再多也不能突破“每线程至少 16 桶”的分块粒度带来的取整损耗，实测加速比只会低于线程数之比。

### 10.5 并发度：从 16 段到“桶数个写槽位”

- JDK 7：并发写槽位数 = 段数，默认 16；两个线程落同一段即串行等待。
- JDK 8：理论写槽位数 ≈ 桶数 = 表大小，表每翻倍一次，槽位翻倍：表 16 → 16 个，表 64 → 64 个，表 256 → 256 个。
- 热点降级换算（哈希均匀假设）：两个写线程落同一桶的概率 = 1 / 桶数：16 桶时 1 / 16 = 6.25%，64 桶时 1 / 64 ≈ 1.56%，256 桶时 1 / 256 ≈ 0.39%。
- 16 个写线程同时写不同 key、至少一对撞桶的概率（生日悖论式换算，均匀哈希假设）：表 16 时 ≈ 100%（99.9999%），表 64 时 87.10%，表 256 时 38.03%。即表 16 时 16 个并发写几乎必然有人撞桶排队；表扩到 256 后仍有约 38% 的概率至少撞一次——“桶数 = 并发度”只是上限，真实并行度还要看哈希分布。
- 计数分片不重算：CounterCell 分散计数的 Cell 数上限换算见 CAS 与原子类篇 LongAdder 一节，本节只用其结论（`size()` 为近似值），不重复换算。

### 10.6 边界条件与换算假设

- 全部换算默认哈希均匀、负载因子 0.75；哈希函数差、key 的 hashCode 扎堆时，单桶链长与热点概率都会高于本节算式，泊松概率只作量级参考。
- 树化概率用泊松近似：真实分布受扰动函数 `spread` 与 key 分布影响，1.28 × 10^(−6) 是 λ = 0.75 下的理论值，不是实测值。
- transfer 步长口径以 OpenJDK JDK 8 源码为准（(n >>> 3) / NCPU，下限 16）；不同 JDK 大版本或小版本实现可能有差异，JDK 7 没有 transfer 协助机制。
- 耗时对比假设每桶搬移耗时相同、忽略线程争用、调度与 ForwardingNode 协助开销，是理想上限；实测加速比低于线程数之比。
- `size()` 与 `mappingCount()` 均为近似值，不能用本节的阈值换算反推精确元素数；精确计数需另维护 LongAdder，见第 4.4 节与 CAS 与原子类篇。
- JDK 7 分段锁的数字只作历史对照：JDK 8+ 已无 Segment，16 段的并发度结论不适用于 JDK 8+。

## 小技巧与冷知识

【小技巧】`compute` 的重映射函数返回 null 会直接删掉这个 key：`map.compute(k, (key, val) -> null)` 等价于一次原子的 remove。想“满足条件才删”时别写先 get 再 remove 的两步，用 compute 一步完成“读-判断-删”，判断和删除之间别的线程插不进来；返回 null 不是“算出个空值”，而是明确的删除语义。

【小技巧】`computeIfAbsent` 的映射函数返回 null 时不建立任何映射：函数算出 null，map 里什么都不放，下次再调同一个 key 还会重新算一次。别拿“返回 null”当“已经算过、结果是空”的标记——缓存场景里这意味着缓存永远不命中，函数会被反复调用；需要缓存“空结果”时用一个专门的哨兵对象当 value。

【小技巧】需要“并发 Set 但又要随时查 key 对应的值”时，用带默认值的 `keySet` 视图：`map.keySet(defaultValue)` 返回的 Set 里每 add 一个 key，就等于往 map 里放 `key -> defaultValue`。`newKeySet()` 丢掉了 value 这一侧，带默认值的 keySet 两边都保住了，Set 的 add/remove 和 Map 的读写是同一份数据。

【小技巧】别拿 `containsValue` 做高频判断：它是 O(n) 全表扫描，元素一多就慢；而且结果是弱一致的——扫描中途别的线程改了值，返回的结果可能既不是扫描开始前的值，也不是扫描结束后的值。业务里需要“按值反查 key”时，另外建一个反向索引 Map，列表式扫描只适合偶尔一次的排查。

【小技巧】并行 `search` 会短路，但返回的是“先算出来的那个”，不是“遇到的第一个”：`search(threshold, (k, v) -> 匹配 ? v : null)` 一旦某个线程找到非 null 结果就尽早返回，不用等全表扫完。哪个线程先找到是不确定的，多个匹配时别指望它按 key 顺序返回第一个；要确定性顺序就先把候选集收拢再在单线程里挑。

【小技巧】别在 ConcurrentHashMap 实例上加 `synchronized` 指望复合操作变原子：它的内部方法根本不在 map 对象上同步，锁只落在单个桶头节点上，外部 `synchronized(map)` 锁不住 put/compute 的内部路径。复合操作要么用 compute/merge 系方法一步完成，要么自己另加一把读写锁把整个复合操作包起来，锁 map 对象本身是白锁。

【小技巧】把它当缓存用必须自己管上限：ConcurrentHashMap 没有任何淘汰机制，只进不出，key 一直涨下去最后就是 OOM。需要容量上限、过期、LRU 淘汰时直接用 Caffeine 或 Guava Cache，别手写定时任务扫全表删——定时清理本身又是一次全表扫描，还会在清理瞬间和读写抢桶锁。

【小技巧】超大表别用 `size()` 拿元素数：`size()` 返回 int，元素数超过 `Integer.MAX_VALUE` 时直接封顶返回它，不会告诉你真实值。`mappingCount()` 返回 long，同样是近似值但不会被 int 上限截断；需要精确计数时这俩都不行，得自己用 LongAdder 另维护一份计数。

【小技巧】`putAll` 整体不是原子的：它只保证每一次 put 各自原子，批量放一半时别的线程已经能看到前一半数据。需要“批量初始化一次可见”时，先在本地 HashMap 里把数据装好，再把整个 map 引用通过 `volatile` 字段或 `AtomicReference` 一次性发布出去；或者把整批数据包成一个整体对象，用 `computeIfAbsent` 占位一个 key。

【冷知识】ConcurrentHashMap 是 Java 5 随 JSR 166 一起进 JDK 的：它是 `java.util.concurrent` 首批类之一，由 Doug Lea 主导设计，2004 年随 JDK 5 发布。也就是说它比 JDK 8 的 CAS + 桶头锁重写早了整整十年，最初的版本是 JDK 7 那套 Segment 分段锁结构。

【冷知识】源码里桶头节点的 hash 有三个特殊负值，正常 hash 永远是非负数：`MOVED = -1`（ForwardingNode，桶已迁移）、`TREEBIN = -2`（红黑树根节点）、`RESERVED = -3`（computeIfAbsent 占位节点）。`spread` 扰动函数会把 key 的 hashCode 最高位置 0，保证正常节点的 hash 都是正数；读源码时看到负 hash，一眼就能认出这是特殊节点而不是真实数据。

【冷知识】JDK 自己也在用 ConcurrentHashMap：`java.lang.ClassLoader` 的并行类加载锁表 `parallelLockMap` 就是一个 ConcurrentHashMap，按类名给每个正在加载的类配一把锁。类加载是典型的高并发读场景——同一个类可能被多个线程同时请求加载，JDK 没另写一套并发结构，直接复用了它。

## 11. 面试与自测

1. 为什么不允许 null key / null value？为什么 containsKey 也救不了？
2. JDK 7 分段锁并发度为什么是 16？改大有什么代价？
3. JDK 8 为什么用 synchronized 锁桶头，而不是 ReentrantLock？
4. put 时桶为空和桶不为空，分别走什么路径？
5. ForwardingNode 是什么？读写线程遇到它会怎么做？
6. size() 为什么不准？要精确计数的代价是什么？
7. computeIfAbsent 里调用 put 同一个 map 会怎样？
8. 它与 Hashtable、synchronizedMap 的迭代器行为有什么区别？
- 参考要点：见第 1、2、4、6 节，能画出扩容协助流程即过关。

9. 预期要放 10000 个元素，直接用默认构造器从 16 起步与按预期容量初始化，扩容次数与累计搬移量各差多少？并说明初始表大小与阈值怎么算。

### 参考答案

1. 禁 null 是为消除歧义：并发下 get(k) 返回 null 无法区分“key 不存在”还是“key 存在但 value 就是 null”。containsKey 也救不了：containsKey 与 get 之间别的线程可能插入或删除该 key，两次调用之间没有原子性，歧义仍在；所以干脆禁掉 null，放入直接抛 NullPointerException。
2. 默认 16 段：Segment 数组默认长度 16，每段一把 ReentrantLock，并发度 = 段数，理想情况下最多 16 个线程同时写不同段。改大的代价：段数在构造时定死，扩容只扩单段、不扩段数；段数越大内存开销越大（每段一个锁 + 一个 HashEntry 数组），且并发度仍被构造时的段数封顶。
3. 桶头锁生命周期极短，JVM 对 synchronized 的优化已足够；更重要的是省内存——桶数随容量增长，每个桶配一个 ReentrantLock 对象开销太大，而 synchronized 的锁状态记在对象头 Mark Word 里。JDK 8 锁粒度从“一段”细到“一桶”，冲突概率大幅降低。
4. 桶为空：CAS 无锁插入（casTabAt）。桶不为空：synchronized 锁住桶头节点，遍历链表/红黑树插入或更新；若桶是 ForwardingNode（正在扩容），先协助扩容再重试；空表时先初始化。
5. ForwardingNode 是已迁移桶的标记（hash = MOVED = -1）。读写线程命中它时不等待，直接调用 helpTransfer 协助搬剩余桶，搬完再到新表操作；transfer 按步长分工，默认每线程至少负责 16 个桶。
6. size() 不准：计数是 baseCount + CounterCell[] 分散累加，sum() 时各分片不在同一时刻统计，读取瞬间还可能有更新在途，所以是估算值。精确计数的代价：要全表加锁或让写停下来等计数静止，等于放弃并发，违背 CHM 的设计目标；mappingCount() 同样是近似值。
7. 映射函数执行时持有桶锁，函数里再 put 同一个 map：同一 key 递归回写会等同一把锁，直接卡死或抛 IllegalStateException: Recursive update；不同 key 也可能拖住同桶其他 key。解法：映射函数只做纯计算，依赖数据提前在锁外算好再传入。
8. Hashtable 的 Enumeration/迭代器是弱一致性；synchronizedMap 的迭代器是快速失败，迭代需手动同步全表；CHM 的迭代器是弱一致性：不抛 ConcurrentModificationException，迭代中允许其他线程增删，看到的是某一时刻的快照式视图，可能漏掉新插入的，也可能看到刚被删的。

9. 预期一万个元素时，单参构造器按一万的一点五倍加一得一万五千零一，再向上取二的幂得一万六千三百八十四，即初始表大小，阈值是一万两千二百八十八；从默认十六起翻倍扩容十次，累计搬移一万六千三百六十八个桶，远多于直接按预期容量初始化的一次建表。

## 12. 关联知识

- HashMap：单线程结构原型，数组 + 链表 + 红黑树、扰动函数、扩容机制先看它。
- Java-并发编程：CAS、volatile 可见性、synchronized、LongAdder 分散计数思想。
- Hashtable：全表 synchronized 的老方案，理解它才知道分段锁解决了什么。
- LongAdder / AtomicLong：计数类需求的专用工具，与 CounterCell 同源思想。
