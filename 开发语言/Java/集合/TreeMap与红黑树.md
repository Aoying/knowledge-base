---
知识点: TreeMap与红黑树
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - HashMap
  - LinkedHashMap与LRU
标签:
  - Java
  - 集合
  - 数据结构
---

# TreeMap与红黑树

## 1. 核心概念

- `TreeMap` 是基于红黑树的有序 `Map`，实现 `NavigableMap`：所有 key 按序排列，遍历即有序。
- 排序依据二选一：key 的自然顺序（key 实现 `Comparable`），或构造时传入的 `Comparator`，后者优先。
- `put` / `get` / `remove` 都是 O(log n)，没有 [HashMap](开发语言/Java/集合/HashMap.md) 的 O(1)，换来的是顺序与范围查询能力。
- 非线程安全；自然顺序下不允许 `null` key（比较时会抛 `NullPointerException`）。

## 2. 基础知识

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(30, "c"); map.put(10, "a"); map.put(20, "b");
map.keySet();            // 10, 20, 30，与插入顺序无关
map.firstKey();          // 10
map.lastKey();           // 30
map.floorKey(25);        // ≤25 的最大 key：20
map.ceilingKey(25);      // ≥25 的最小 key：30
map.higherKey(20);       // 严格大于 20：30
map.pollFirstEntry();    // 弹出最小键值对
```

### 范围视图（活视图，不是拷贝）

```java
map.subMap(10, 25);      // [10, 25) 区间的视图
map.headMap(20);         // <20 的视图
map.tailMap(20);         // ≥20 的视图
map.descendingMap();     // 逆序视图
```

- 视图与原 map 双向联动：改视图影响原 map，反之亦然。
- 向视图里 put 区间外的 key 抛 `IllegalArgumentException`，这是最常见的范围视图坑。

## 3. 进阶知识

### Comparator 与 equals 一致性

- TreeMap 判断「同一个 key」只看比较结果是否为 0，**不看 equals**。
- 《Map》接口建议比较与 equals 保持一致，但不强制。不一致时的典型事故：

```java
// 按字符串长度比较：长度相同即视为同一 key
TreeMap<String, Integer> m = new TreeMap<>(Comparator.comparingInt(String::length));
m.put("ab", 1);
m.put("cd", 2);   // 比较结果 0，被当成同一 key，覆盖为 2，且 key 仍是 "ab"
```

- 另两个比较器坑：比较逻辑用了可变字段，put 之后字段变化，树的有序性被破坏，后续 get/remove 按新值找旧节点会失败；比较器不满足传递性（a<b、b<c、c<a），树结构会被悄悄插乱且不报任何异常，只在查不到数据时暴露。

## 4. 原理剖析

### 红黑树的五条性质

1. 节点非红即黑；
2. 根节点黑色；
3. 叶子（NIL 空节点）黑色；
4. 红节点的两个子节点都是黑（红不能相邻）；
5. 任一节点到其所有叶子路径的黑节点数相同。

- 由 4、5 推出：最长路径（红黑交替）不超过最短路径（全黑）的 2 倍，树高 ≤ 2·log(n+1)，所以增删查稳定 O(log n)，不会退化成链表。
- 它是「近似平衡」而非严格平衡：比 AVL 树旋转次数少，插入删除更便宜，适合写多读多的通用容器——这也是 HashMap 冲突树化、以及各类语言有序容器普遍选红黑树的原因。

### 实现要点

- 节点 `Entry` 含 `key`、`value`、`left`、`right`、`parent`、`color` 五个字段，父引用让插入后回溯修复（变色 + 左右旋）不需要栈。
- 插入新节点默认涂红（对性质 5 无影响），再按叔叔节点颜色分情况修复；删除修复更复杂，是源码里最长的一段。
- 注意区分：HashMap 树化用的 `TreeNode` 是另一套独立实现，只是同为红黑树思想，不要混为一谈（`TreeNode` 继承自 [LinkedHashMap](开发语言/Java/集合/LinkedHashMap与LRU.md) 的 Entry）。

## 5. 实战应用

- 典型场景：按时间/价格/分数的区间统计、排行榜取 Top N 前后邻居、需要 floor/ceiling 语义的配置分段（如按分值查档位）。
- 选型对比：只要顺序不要范围查询，用 [LinkedHashMap与LRU](开发语言/Java/集合/LinkedHashMap与LRU.md)（顺序是插入/访问序）；key 无序且量大，用 HashMap；并发有序需求用 `ConcurrentSkipListMap`（跳表实现，弱一致）。
- 构造带比较器的 TreeMap 后，比较器就是这份数据的「身份定义」，序列化、跨服务传递时要把比较规则一并说清。

## 6. 疑难杂症

### 场景 1：put 进去却 get 不到

- 排查：key 是否可变且比较字段被改过；比较器是否对两个不同 key 返回了 0；自然顺序的 key 类 `compareTo` 是否与预期一致（如大小写、数值类型混用）。
- 定位手法：同一数据灌进 HashMap 能 get 到、TreeMap 不能，问题必在比较逻辑。

### 场景 2：遍历顺序「差不多对」但个别元素乱序

- 多半是比较器不满足传递性或全序要求（如只比了高位字段、浮点数直接相减溢出）。整数比较用 `Integer.compare`，不要写 `a - b`。

## 7. 常见误区

- 误区：TreeMap 是 HashMap 的有序版平替。复杂度从 O(1) 变 O(log n)，无序场景白白变慢。
- 误区：subMap 是拷贝。实为活视图，改动互相影响，且有区间准入限制。
- 误区：比较器返回 0 只是「相等」。在 TreeMap 里它就是同一 key，会覆盖。

## 8. 性能优化

- 大数据量下 get 的常数比 HashMap 大一个量级；读热点且不需要顺序时别硬用 TreeMap。
- 范围查询用视图或 `floorKey` 等导航方法，不要全表遍历过滤。

## 9. 版本差异

- `NavigableMap` 的 floor/ceiling/poll 等方法自 JDK 6 引入后稳定；JDK 8 起可配合函数式 API 使用，无结构变化。

## 小技巧与冷知识

【小技巧】多级排序用 `Comparator.comparing(...).thenComparing(...)` 链起来，别只比一个字段：只比高位字段（比如只比分数）时，分数相同的不同 key 比较结果是 0，会被当成同一个 key 覆盖掉。链式比较器先比主字段、再比次字段（分数相同比学号），比较结果为 0 才真正意味着“同一个 key”；比较器写几级，取决于“几个字段合起来才能唯一区分一个 key”。

【小技巧】比较浮点数别用减法，用 `Double.compare`：`a - b` 在 double 上会丢精度、还可能溢出成无穷大，NaN 参与减法结果还是 NaN，比较直接乱掉。`Double.compare` 对 NaN 有明确定义——NaN 比任何数都大、两个 NaN 视为相等（比较结果为 0，即同一 key）；比较器里遇到浮点字段，先问一句“NaN 该排哪”，别让它悄悄混进树里。

【小技巧】`floorKey` / `ceilingKey` 查不到时返回 null，不是抛异常：按分值查档位、按时间查最近一次配置时，查不到就是“没有更小/更大的 key”，返回 null 是正常结果。拿到返回值先判 null 再用，别直接解包或调方法；“找不到”在导航方法里是返回值语义，不像 `firstKey` 那样抛异常，两类方法的空处理方式不一样。

【小技巧】取 Top N 别全表排序，用 `descendingMap` 迭代取前 N 个：排行榜取前 10 名时，`map.descendingMap().entrySet()` 从最大往下走，数够 10 个就停，复杂度是 O(N log n) 级别而不是全表遍历加排序。反过来从最小取前 N 个直接正序迭代就行；“取一端的前几名”是 TreeMap 的主场，全表遍历过滤是把它当 HashMap 用。

【小技巧】`subMap(from, to)` 默认左闭右开，右端点想含进来要用四参数版：`subMap(10, 25)` 取的是 [10, 25)，25 本身不在区间里；想含 25 得写 `subMap(10, true, 25, true)`。区间边界含不含是范围查询最常见的差一错误，写视图时先把四个布尔参数想清楚——`headMap(to, true)`、`tailMap(from, false)` 同理，别靠“试一次看结果”定边界。

【小技巧】遍历 TreeMap 时别在循环里调 `map.put` / `remove`，要用迭代器的 `remove()`：增强 for 内部就是迭代器，循环体里直接改 map 会触发 fail-fast 抛 `ConcurrentModificationException`，视图遍历也一样。需要边遍历边删时显式拿 `Iterator`、调 `it.remove()`；需要边遍历边加时先收集到临时集合，遍历结束再统一 put——“遍历中改结构”是所有集合的通病，TreeMap 也不例外。

【小技巧】自定义 key 的 `compareTo` 要和 `equals` 保持一致，且比较字段要不可变：比较结果为 0 但 `equals` 为 false 时，`containsKey` 和 `containsValue` 会给出互相矛盾的答案，序列化后重建的 map 也可能“丢”数据。比较字段用 `final` 修饰、put 之后不再改；字段一改，节点在树里的位置就错了，get 按新值找旧节点必然失败——这条和原文比较器坑是同一根子，落点在“字段本身就不能变”。

【小技巧】想支持 null key 时用 `Comparator.nullsFirst` / `nullsLast` 包装：自然顺序下 null key 无法比较直接抛 NPE，但自定义比较器可以显式处理 null——`new TreeMap<>(Comparator.nullsFirst(Comparator.naturalOrder()))` 让 null 排在最前，`nullsLast` 让它排最后。null 的位置是比较器说了算，不是“放不放得进”的问题；不包装就别放 null，别等线上 NPE 再想起这条。

【小技巧】`descendingMap().descendingMap()` 回到的是原 map，不用拷全表建逆序副本：`descendingMap` 返回的是逆序活视图，对它再调一次 `descendingMap`，拿到的就是原顺序的视图本身。想在“正序处理”和“逆序处理”之间来回切时，直接在两个视图间切换；新建一个 `new TreeMap<>(Comparator.reverseOrder())` 再 putAll 全表，是白白多一次 O(n log n) 的拷贝。

【小技巧】多线程下用 TreeMap 要么包 `Collections.synchronizedMap`、要么换 `ConcurrentSkipListMap`：`synchronizedMap` 只锁单个方法调用，遍历时仍需手动 `synchronized (map)` 包住整个循环，否则遍历中途被改照样抛异常。读多写少且要范围查询的有序并发场景，直接用跳表实现的 `ConcurrentSkipListMap`，它和 TreeMap 一样实现 `NavigableMap`，导航方法基本能平替。

【小技巧】空 map 上调 `firstKey` / `lastKey` 抛 `NoSuchElementException`，先判空或用 `firstEntry`：`firstKey` 在没有元素时没有“返回 null”这个选项，只能抛异常；`firstEntry` / `lastEntry` 在空 map 上返回 null。取两端元素前要么先 `isEmpty()`，要么用返回 null 的 Entry 版——“取不到”在 TreeMap 里有抛异常和返回 null 两种风格，用错了就是一次不必要的崩溃。

【冷知识】红黑树最早不叫红黑树：Rudolf Bayer 在 1972 年发明时叫它“对称二叉 B 树”（symmetric binary B-tree），Leonidas Guibas 和 Robert Sedgewick 在 1978 年的论文里把它改名成 red-black tree，红黑只是给节点涂的两种颜色，名字本身没有更深的含义。

【冷知识】`TreeSet` 内部就是一个 `TreeMap`：和 `HashSet` 内部包一个 `HashMap` 一模一样，`TreeSet` 的每个元素在内部 map 里对应一个 value 为固定占位对象 `PRESENT` 的键值对。理解了这层关系就知道 `TreeSet` 的排序、范围查询、比较器坑和 `TreeMap` 是同一套——它只是把 value 藏起来的 TreeMap。

【冷知识】Linux 内核的完全公平调度器（CFS）也用红黑树：内核按每个进程的虚拟运行时间（vruntime）把可运行进程排进一棵红黑树，调度时取最左边的节点（vruntime 最小者）运行。红黑树在这里的价值和 TreeMap 一样——有序、增删查 O(log n)、不需要严格平衡；操作系统调度器和 Java 集合用的是同一种数据结构。

## 10. 面试与自测

- 红黑树五条性质是什么？为什么树高是 O(log n)？
- 为什么通用有序容器选红黑树而不是 AVL 树？
- TreeMap 判断 key 相同看什么？与 equals 不一致会出什么事？
- subMap/headMap 是视图还是拷贝？视图写入有什么限制？
- 自然顺序下为什么不能放 null key？
- 自测：用「按长度比较」的比较器复现 key 被覆盖，再改成长度+字典序两级比较修复。

### 参考答案

1. 五条性质：节点非红即黑；根节点黑色；叶子（NIL 空节点）黑色；红节点的两个子节点都是黑（红不能相邻）；任一节点到其所有叶子路径的黑节点数相同。树高 O(log n)：由红不相邻 + 黑高相同推出最长路径不超过最短路径的 2 倍，树高 ≤ 2·log(n+1)，增删查稳定 O(log n)，不会退化成链表。
2. 红黑树是近似平衡而非严格平衡：比 AVL 树旋转次数少，插入删除更便宜，适合写多读多的通用容器；AVL 严格平衡、读更快但写旋转更多。HashMap 冲突树化、各类语言有序容器普遍选红黑树，都是这个权衡。
3. TreeMap 判断 key 相同只看比较结果是否为 0，不看 equals。不一致的事故：按字符串长度比较时，“ab”与“cd”长度相同即视为同一 key，后 put 的覆盖前一个，且 key 仍是“ab”；另两个坑：比较字段可变会破坏树有序性，比较器不满足传递性会悄悄插乱树结构且不报错。
4. subMap/headMap/tailMap/descendingMap 都是活视图，不是拷贝：改视图影响原 map，反之亦然。视图写入限制：向视图里 put 区间外的 key 抛 IllegalArgumentException。
5. 自然顺序下 null key 无法比较：没有 Comparator 时靠 key 的 Comparable，自然顺序比较 null 会抛 NullPointerException；所以自然顺序下不允许 null key（自定义 Comparator 对 null 有处理时另说）。
6. 自测思路：用“按长度比较”的比较器 put “ab” 再 put “cd”，复现 key 被覆盖（size 仍为 1，key 仍是“ab”）；改成长度 + 字典序两级比较修复。验证要点：修复后“ab”与“cd”共存、size 为 2；同时确认比较器满足传递性，整数比较用 Integer.compare、不要写 a - b。

## 11. 关联知识

- [HashMap](开发语言/Java/集合/HashMap.md)：无序 O(1) 的对照组；树化思想同源但实现独立。
- [LinkedHashMap与LRU](开发语言/Java/集合/LinkedHashMap与LRU.md)：同样「有序」，一个按值排序，一个按插入/访问排序。
- [Queue与Deque](开发语言/Java/集合/Queue与Deque.md)：`PriorityQueue` 是堆实现的有序队列，与 TreeMap 的全序维护成本不同。
