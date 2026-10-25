---
知识点: ConcurrentHashMap
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-22
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

## 10. 面试与自测

1. 为什么不允许 null key / null value？为什么 containsKey 也救不了？
2. JDK 7 分段锁并发度为什么是 16？改大有什么代价？
3. JDK 8 为什么用 synchronized 锁桶头，而不是 ReentrantLock？
4. put 时桶为空和桶不为空，分别走什么路径？
5. ForwardingNode 是什么？读写线程遇到它会怎么做？
6. size() 为什么不准？要精确计数的代价是什么？
7. computeIfAbsent 里调用 put 同一个 map 会怎样？
8. 它与 Hashtable、synchronizedMap 的迭代器行为有什么区别？
- 参考要点：见第 1、2、4、6 节，能画出扩容协助流程即过关。

### 参考答案

1. 禁 null 是为消除歧义：并发下 get(k) 返回 null 无法区分“key 不存在”还是“key 存在但 value 就是 null”。containsKey 也救不了：containsKey 与 get 之间别的线程可能插入或删除该 key，两次调用之间没有原子性，歧义仍在；所以干脆禁掉 null，放入直接抛 NullPointerException。
2. 默认 16 段：Segment 数组默认长度 16，每段一把 ReentrantLock，并发度 = 段数，理想情况下最多 16 个线程同时写不同段。改大的代价：段数在构造时定死，扩容只扩单段、不扩段数；段数越大内存开销越大（每段一个锁 + 一个 HashEntry 数组），且并发度仍被构造时的段数封顶。
3. 桶头锁生命周期极短，JVM 对 synchronized 的优化已足够；更重要的是省内存——桶数随容量增长，每个桶配一个 ReentrantLock 对象开销太大，而 synchronized 的锁状态记在对象头 Mark Word 里。JDK 8 锁粒度从“一段”细到“一桶”，冲突概率大幅降低。
4. 桶为空：CAS 无锁插入（casTabAt）。桶不为空：synchronized 锁住桶头节点，遍历链表/红黑树插入或更新；若桶是 ForwardingNode（正在扩容），先协助扩容再重试；空表时先初始化。
5. ForwardingNode 是已迁移桶的标记（hash = MOVED = -1）。读写线程命中它时不等待，直接调用 helpTransfer 协助搬剩余桶，搬完再到新表操作；transfer 按步长分工，默认每线程至少负责 16 个桶。
6. size() 不准：计数是 baseCount + CounterCell[] 分散累加，sum() 时各分片不在同一时刻统计，读取瞬间还可能有更新在途，所以是估算值。精确计数的代价：要全表加锁或让写停下来等计数静止，等于放弃并发，违背 CHM 的设计目标；mappingCount() 同样是近似值。
7. 映射函数执行时持有桶锁，函数里再 put 同一个 map：同一 key 递归回写会等同一把锁，直接卡死或抛 IllegalStateException: Recursive update；不同 key 也可能拖住同桶其他 key。解法：映射函数只做纯计算，依赖数据提前在锁外算好再传入。
8. Hashtable 的 Enumeration/迭代器是弱一致性；synchronizedMap 的迭代器是快速失败，迭代需手动同步全表；CHM 的迭代器是弱一致性：不抛 ConcurrentModificationException，迭代中允许其他线程增删，看到的是某一时刻的快照式视图，可能漏掉新插入的，也可能看到刚被删的。

## 11. 关联知识

- HashMap：单线程结构原型，数组 + 链表 + 红黑树、扰动函数、扩容机制先看它。
- Java-并发编程：CAS、volatile 可见性、synchronized、LongAdder 分散计数思想。
- Hashtable：全表 synchronized 的老方案，理解它才知道分段锁解决了什么。
- LongAdder / AtomicLong：计数类需求的专用工具，与 CounterCell 同源思想。
