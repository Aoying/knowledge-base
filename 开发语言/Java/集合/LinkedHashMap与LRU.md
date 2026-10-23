---
知识点: LinkedHashMap与LRU
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-23
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

## 10. 面试与自测

- LinkedHashMap 怎么实现顺序？两种顺序模式的区别？
- `removeEldestEntry` 在什么时候被调用？为什么一次 put 最多淘汰一个？
- 访问顺序模式下 get 为什么算结构性修改？会引发什么异常？
- 手写一个容量 3 的 LRU，依次 put 1、2、3、get 1、put 4，手推最终内容与顺序。
- 它和 [TreeMap](开发语言/Java/集合/TreeMap与红黑树.md) 都能「有序」，排序依据有什么本质区别？

## 11. 关联知识

- [HashMap](开发语言/Java/集合/HashMap.md)：父类，桶结构、扩容、树化都看它。
- [TreeMap与红黑树](开发语言/Java/集合/TreeMap与红黑树.md)：按 key 值排序的有序 Map，与按访问/插入顺序是两种思路。
- [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md)：fail-fast 的 modCount 机制同源。
