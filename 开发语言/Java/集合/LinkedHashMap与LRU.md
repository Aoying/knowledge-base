---
知识点: LinkedHashMap与LRU
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - HashMap
  - TreeMap与红黑树
  - ArrayList与扩容
标签:
  - Java
  - 集合
  - 缓存
---

# LinkedHashMap与LRU

## 1. 核心概念

- `LinkedHashMap` 继承 [HashMap](开发语言/Java/集合/HashMap.md)，在哈希表之上再维护一条双向链表，记录节点的先后关系。
- 两种顺序模式，由构造参数 `accessOrder` 决定：
  - `false`（默认）：插入顺序，先 put 的在前；重复 put 已有 key 不改变位置。
  - `true`：访问顺序，最近被 `get` / `put` 的挪到链尾，链头就是最久未访问的——LRU 的基础。
- 性能与 HashMap 基本一致，代价是每个节点多两个引用、增删时多维护链表。非线程安全。

## 2. 基础知识

```java
// 插入顺序：输出 a, b, c
Map<String, Integer> ins = new LinkedHashMap<>();
ins.put("a", 1); ins.put("b", 2); ins.put("c", 3);
ins.keySet().forEach(System.out::println);

// 访问顺序：get("a") 后 a 挪到链尾，输出 b, c, a
Map<String, Integer> acc = new LinkedHashMap<>(16, 0.75f, true);
acc.put("a", 1); acc.put("b", 2); acc.put("c", 3);
acc.get("a");
acc.keySet().forEach(System.out::println);
```

- 需要「按插入顺序遍历」时用它替代 HashMap，成本极低：配置项顺序、表单字段顺序、按添加次序输出的报表都适合。

## 3. 进阶知识

### 钩子方法：HashMap 预留的回调

`HashMap` 的增删流程里埋了三个空方法，`LinkedHashMap` 重写它们维护链表：

| 回调 | 触发时机 | LinkedHashMap 的动作 |
|---|---|---|
| `afterNodeAccess(e)` | get 命中、put 更新已有 key 后 | 访问顺序模式下把节点移到链尾 |
| `afterNodeInsertion(evict)` | 插入新节点后 | 调用 `removeEldestEntry`，返回 true 就删链头 |
| `afterNodeRemoval(e)` | 删除节点后 | 把节点从双向链表摘除 |

- 插入顺序模式下更新已有 key 不触发移位，所以顺序稳定；访问顺序模式下连 `get` 都算结构性修改（见第 6 节坑）。

### 手写 LRU 缓存

```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    LruCache(int capacity) {
        super(16, 0.75f, true); // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // 超容就淘汰链头（最久未访问）
    }
}
```

- `removeEldestEntry` 只在插入新节点后被调用一次，所以一次 put 最多淘汰一个，逻辑简单可靠。
- 这正是 Redis 的 `allkeys-lru` 想达到的效果，只是 Redis 用采样近似 LRU 换性能，单机版是精确 LRU（Redis 策略详见缓存相关笔记，此处不展开）。

## 4. 原理剖析

- 节点类型 `LinkedHashMap.Entry` 继承 `HashMap.Node`，多 `before`、`after` 两个引用；全表节点串成一条双向链表，`head` 是最旧端，`tail` 是最新端。
- 哈希表部分（桶数组、扰动、扩容、树化）完全复用 HashMap；扩容只搬桶数组，不影响链表顺序。
- 一个细节：`HashMap.TreeNode` 正是继承自 `LinkedHashMap.Entry`，因为树化节点也要能挂进这条顺序链。

## 5. 实战应用

- 本地小缓存：数据量小、单机、无过期时间要求时，LRU 版 LinkedHashMap 是零依赖方案。
- 线程安全处理：本身不安全，多线程下用 `Collections.synchronizedMap` 包一层（粗粒度锁）；并发量大或需要过期、统计、异步刷新，直接上 Caffeine，不要硬扛。
- 防御性使用：作为缓存必须给容量上限；无淘汰策略的 LinkedHashMap 当缓存用等于内存泄漏。

## 6. 疑难杂症

### 场景 1：访问顺序模式下遍历时 get，抛 ConcurrentModificationException

- 原因：accessOrder=true 时 `get` 会把节点移到链尾，属于结构性修改，`modCount` 递增，正在进行的迭代器校验失败。插入顺序模式下 get 则安全。
- 修复：遍历期间不要 get；需要边遍历边查时先复制一份 key 列表。

### 场景 2：LRU 不淘汰，内存一直涨

- 排查顺序：`accessOrder` 是否传了 true（传 false 也能「淘汰」，但淘汰的是最先插入的 FIFO，不是 LRU）；`removeEldestEntry` 是否重写且返回条件正确；是否只 put 不经过插入路径（如批量 putAll 的淘汰按每个新节点逐个判断，没有问题）。

## 7. 常见误区

- 误区：LinkedHashMap 默认就是 LRU。默认是插入顺序，LRU 要构造时第三参传 true 且重写淘汰方法。
- 误区：覆盖已有 key 会刷新它在插入顺序里的位置。不会，只有访问顺序模式才移位。
- 误区：手写 LRU 可以平替专业缓存。它没有过期、并发、统计、穿透保护，只适合简单场景。

## 8. 性能优化

- 链表维护是常数级开销，热点路径上与 HashMap 差距可忽略；真在意时先 profile，不要预先优化。
- 初始容量按预计元素量给，减少扩容，公式与 HashMap 相同：（预计数 / 0.75）+ 1。

## 9. 版本差异

- 机制自 JDK 1.4 引入后基本稳定，无显著版本差异；JDK 8 起可直接用 HashMap 的函数式 API（`computeIfAbsent` 等），访问顺序模式下它们同样触发节点移位。

## 小技巧与冷知识

【小技巧】想“摸一下”某个 key 让它变新，别用 `containsKey` 或 `getOrDefault`：访问顺序下只有 `get()` 会把节点挪到链尾，`containsKey` 和 `getOrDefault` 内部直接调 `getNode`、不经过刷新顺序的钩子，查了等于没查。判断“有没有”用 `containsKey`、想“刷新访问时间”用 `get`，两个意图别用同一个方法混着办，缓存里“查一下在不在”顺手把热度刷掉是最常见的误用。

【小技巧】`put` 已有 key 也算一次访问：访问顺序模式下更新已有 key 的值，节点同样会被挪到链尾——`putVal` 里找到旧节点后会调刷新顺序的钩子。想显式“touch”某个 key 又不想读它的值时，`put(key, 同样的值)` 就能把它变新；反过来，别以为“只改值不算访问”，覆盖写一样会打乱淘汰顺序。

【小技巧】淘汰只在插入新 key 时触发，更新已有 key 不会淘汰：`removeEldestEntry` 只在 `afterNodeInsertion` 里被调用，`put` 已有 key、哪怕 size 早已超容，也不会触发任何淘汰。超容状态只能由“插入一个新 key”终结——一次插入最多淘汰一个，所以批量 `putAll` 插 10 个新 key 可能连续淘汰 10 个，别指望一次 put 把账算清。

【小技巧】想拿“最久未访问的那一项”，直接取迭代器的第一个元素：访问顺序下链头就是最久未访问的，`map.entrySet().iterator().next()` 拿到的就是它，不用自己再维护时间戳。取的时候用 `entry.getValue()` 读值，别用 `map.get(key)`——后者会把这个刚要淘汰的节点又挪到链尾，查一次就把它“救活”了。

【小技巧】`removeEldestEntry` 里可以顺手做淘汰的收尾工作：参数 `eldest` 就是即将被删的那个节点，需要关连接、记淘汰日志、回写脏数据时，在返回 true 之前用它把事办了，别等事后去日志里反推“刚才删了谁”。但别在里面再 `put`/`get` 同一个 map——淘汰钩子正跑在插入流程中间，重入会把链表和 `modCount` 搅乱。

【小技巧】容量为 0 的 LRU 缓存永远是空的：`removeEldestEntry` 的判断是 `size() > capacity`，capacity 传 0 时每插入一个新节点、size 变成 1 就立即触发淘汰，刚插进去的那个就是链头、马上被删掉。容量至少从 1 开始才有意义；想表达“禁用缓存”就直接别建这个对象，别拿 capacity=0 当开关。

【小技巧】给 LRU 加过期时间时，从链头开始扫最便宜：访问顺序下链头是最旧的，过期项一定集中在链头一侧。定时清理时从 `iterator().next()` 开始往后删，遇到第一个未过期的就停——不用全表扫描。`removeEldestEntry` 只看 size 不看时间，过期逻辑要自己补，这条“从最旧端扫、见新即停”是手写过期缓存最省的写法。

【小技巧】`accessOrder` 只能在构造时定，之后改不了：构造参数第三个 `true/false` 一旦传进去，这张表的顺序模式就定死了，没有 setter 能切换。想把一张插入顺序的表改成 LRU，只能新建一张访问顺序的表再 `putAll` 拷过去；建表之前先问一句“以后要不要按访问时间淘汰”，省得上线后再返工。

【小技巧】`get` 返回 null 时分不清“键不存在”还是“值就是 null”：LinkedHashMap 允许 null 值，缓存里 `get` 到 null 可能是没命中、也可能是命中了一个 null 值。区分要用 `containsKey`，但注意它不刷新访问顺序——“先 containsKey 判存在、再 get 取值”会查两次桶，热 key 上这点开销可以接受，别为了省一次查找把 null 值和缺失混成一回事。

【小技巧】访问顺序模式下 `get` 是写操作，别把它当纯读用：`get` 命中会改链表、递增 `modCount`，`Collections.synchronizedMap` 包一层后，读多写少的场景锁竞争比 HashMap 更重。并发量一上来就别硬扛手写 LRU，直接上 Caffeine；单机小缓存也要注意，遍历时同样要手动加同步块，包装器只锁单个方法调用。

【冷知识】Android 的 `LruCache` 就是基于 LinkedHashMap 实现的：`android.util.LruCache` 内部用 `LinkedHashMap(0, 0.75f, true)`，重写 `removeEldestEntry`、按 `sizeOf()` 统计的字节数而不是条数来淘汰。手机端图片缓存这套写法用了十几年——手写 LRU 不是玩具，Android 官方缓存就是这么搭出来的。

【冷知识】Guava Cache、Caffeine 的淘汰是近似 LRU，不是精确 LRU：它们用采样加访问频率加权来挑淘汰对象，换取高并发下的性能；手写 LinkedHashMap 的淘汰是精确的——链头一定是全局最久未访问的那一个。数据量小、单机、要求淘汰结果可预测时，精确 LRU 反而更简单可控；上了规模再换近似算法，是“先精确后近似”，不是“手写不如库”。

【冷知识】LinkedHashMap 是 JDK 里唯一内置“访问即重排”的 Map：HashMap、TreeMap、EnumMap、ConcurrentHashMap 都没有 accessOrder 这个概念，顺序要么是哈希桶顺序、要么是 key 的值排序、要么干脆无序。想用标准库实现“最近最少使用”，LinkedHashMap 是唯一不用自己维护链表的选择——这也是它在 JDK 集合里不可替代的一点。

## 10. 面试与自测

- LinkedHashMap 怎么实现顺序？两种顺序模式的区别？
- `removeEldestEntry` 在什么时候被调用？为什么一次 put 最多淘汰一个？
- 访问顺序模式下 get 为什么算结构性修改？会引发什么异常？
- 手写一个容量 3 的 LRU，依次 put 1、2、3、get 1、put 4，手推最终内容与顺序。
- 它和 [TreeMap](开发语言/Java/集合/TreeMap与红黑树.md) 都能「有序」，排序依据有什么本质区别？

### 参考答案

1. 顺序实现：节点 LinkedHashMap.Entry 继承 HashMap.Node，多 before、after 两个引用，全表节点串成一条双向链表（head 最旧端、tail 最新端）；哈希表部分完全复用 HashMap。两种模式由 accessOrder 决定：false（默认）插入顺序，重复 put 已有 key 不改变位置；true 访问顺序，最近被 get/put 的挪到链尾，链头就是最久未访问的。
2. removeEldestEntry 在插入新节点后被调用（afterNodeInsertion 回调），且一次插入只调用一次，所以一次 put 最多淘汰一个；返回 true 就删链头。LRU 写法：accessOrder = true + 重写该方法为 size() > capacity。
3. 访问顺序模式下 get 会把节点移到链尾，属于结构性修改，modCount 递增；正在进行的迭代器校验失败，抛 ConcurrentModificationException。插入顺序模式下 get 不移位，遍历时 get 安全。修复：遍历期间不要 get，需要边遍历边查时先复制一份 key 列表。
4. 手推：容量 3，put 1、2、3 → 顺序 [1, 2, 3]；get 1 → 1 挪到链尾 → [2, 3, 1]；put 4 → 超容，淘汰链头 2 → 最终内容 [3, 1, 4]，链头 3 最久未访问、链尾 4 最新。
5. 本质区别：TreeMap 按 key 的值排序（自然顺序 Comparable 或构造时 Comparator），顺序与插入/访问无关；LinkedHashMap 按插入顺序或访问顺序排列，与 key 本身的大小无关。只要顺序不要范围查询用 LinkedHashMap；需要按值排序、范围查询用 TreeMap。

## 11. 关联知识

- [HashMap](开发语言/Java/集合/HashMap.md)：父类，桶结构、扩容、树化都看它。
- [TreeMap与红黑树](开发语言/Java/集合/TreeMap与红黑树.md)：按 key 值排序的有序 Map，与按访问/插入顺序是两种思路。
- [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md)：fail-fast 的 modCount 机制同源。
