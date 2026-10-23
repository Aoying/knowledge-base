---
知识点: TreeMap与红黑树
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-23
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

## 10. 面试与自测

- 红黑树五条性质是什么？为什么树高是 O(log n)？
- 为什么通用有序容器选红黑树而不是 AVL 树？
- TreeMap 判断 key 相同看什么？与 equals 不一致会出什么事？
- subMap/headMap 是视图还是拷贝？视图写入有什么限制？
- 自然顺序下为什么不能放 null key？
- 自测：用「按长度比较」的比较器复现 key 被覆盖，再改成长度+字典序两级比较修复。

## 11. 关联知识

- [HashMap](开发语言/Java/集合/HashMap.md)：无序 O(1) 的对照组；树化思想同源但实现独立。
- [LinkedHashMap与LRU](开发语言/Java/集合/LinkedHashMap与LRU.md)：同样「有序」，一个按值排序，一个按插入/访问排序。
- [Queue与Deque](开发语言/Java/集合/Queue与Deque.md)：`PriorityQueue` 是堆实现的有序队列，与 TreeMap 的全序维护成本不同。
