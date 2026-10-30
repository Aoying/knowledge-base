---
知识点: HashMap
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java
  - ConcurrentHashMap
  - Java-并发编程
标签:
  - Java
  - 集合
  - 数据结构
---

# HashMap

## 1. 核心概念

- `HashMap` 是基于哈希表的 `Map` 实现，允许 1 个 `null` key、多个 `null` value。
- 不保证顺序：插入顺序、遍历顺序都可能随扩容变化。
- 非线程安全，多线程并发写必须换 `ConcurrentHashMap` 或外部加锁。
- 核心思路：用 key 的 hash 定位桶，桶内再用 `equals` 精确匹配。

| 操作 | 平均复杂度 | 最坏复杂度（JDK 8） | 说明 |
|---|---|---|---|
| `put` | O(1) | O(log n) | 冲突严重时走红黑树 |
| `get` | O(1) | O(log n) | 链表未树化时最坏 O(n) |
| `remove` | O(1) | O(log n) | 同上 |
| `containsKey` | O(1) | O(log n) | 本质同 `get` |

## 2. 基础知识

### 数据结构：数组 + 链表 + 红黑树

- JDK 8 结构：桶数组 `table` + 链表 + 红黑树。
- 桶内冲突少时用链表，冲突多且容量足够时转红黑树。
- `table` 懒初始化：构造时不分配，第一次 `put` 时才建数组。

```java
// 源码简化：Node 是单链表节点，TreeNode 继承自 LinkedHashMap.Entry
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    V value;
    Node<K, V> next;
}
```

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
Integer v = map.get("a");          // 1
map.remove("b");
map.forEach((k, val) -> System.out.println(k + "=" + val));
```

### 关键参数

| 参数 | 默认值 | 含义 |
|---|---|---|
| 初始容量 `initialCapacity` | 16 | 桶数组长度，必须是 2 的幂 |
| 负载因子 `loadFactor` | 0.75 | 填充比例，空间与冲突的折中 |
| 阈值 `threshold` | 容量 × 负载因子 | `size` 超过它就扩容，16 时为 12 |
| 树化阈值 `TREEIFY_THRESHOLD` | 8 | 桶内链表长度 > 8 时尝试树化 |
| 退化阈值 `UNTREEIFY_THRESHOLD` | 6 | 红黑树节点数 ≤ 6 时退回链表 |
| 最小树化容量 `MIN_TREEIFY_CAPACITY` | 64 | 容量 < 64 时只扩容，不树化 |

- 树化和退化阈值不相等，是为了避免在临界点反复转换。
- 负载因子调大省内存但冲突多，调小查得快但浪费空间，一般不改 0.75。

## 3. 进阶知识

### 函数式 API（JDK 8）

```java
Map<String, Integer> cnt = new HashMap<>();
// 不存在才计算，避免先 get 再 put 两步
cnt.computeIfAbsent("word", k -> 0);
// 原子合并：计数、累加常用
cnt.merge("word", 1, Integer::sum);
cnt.computeIfPresent("word", (k, v) -> v + 1);
cnt.putIfAbsent("word", 0);
int n = cnt.getOrDefault("missing", 0);
```

### key 的约定

- 用作 key 的类必须同时重写 `equals` 和 `hashCode`，且两者用同一批字段。
- 约定：`a.equals(b)` 为 `true` 时，`a.hashCode() == b.hashCode()` 必须成立。
- 反过来不要求：`hashCode` 相同，`equals` 可以不同，那叫哈希冲突。
- key 最好不可变，如 `String`、`Integer`、不可变自定义类。

## 4. 原理剖析

### hash 计算与索引定位

```java
static final int hash(Object key) {
    int h;
    // 高 16 位异或低 16 位：让高位也参与索引计算，减少冲突
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 索引定位
int index = (n - 1) & hash;
```

- 为什么扰动：容量较小时索引只用低位，高位差异被浪费，异或后分布更均匀。
- 为什么容量是 2 的幂：`n` 为 2 的幂时，`(n - 1)` 低位全是 1，`(n-1) & hash` 等价于 `hash % n`，但位运算更快，且分布均匀。
- 构造时传入非 2 的幂容量，会被 `tableSizeFor()` 向上取整到最近的 2 的幂。

### put 流程

1. `table` 为空则先 `resize()` 初始化。
2. 计算 `hash`，用 `(n - 1) & hash` 定位桶。
3. 桶为空：直接插入新 `Node`。
4. 桶不为空且首节点 key 相同（hash 相等且 `equals`）：直接覆盖 value。
5. 首节点是 `TreeNode`：走红黑树插入，平衡后可能调整树结构。
6. 否则走链表：尾插新节点，途中遇到相同 key 就覆盖并结束。
7. 插入后桶内节点数 > 8：调用 `treeifyBin`，容量 < 64 先扩容，否则转红黑树。
8. 插入后 `size > threshold`：扩容。

```java
map.put(k, v); // 覆盖已有 key 时 size 不变，也不触发扩容判断之外的逻辑
```

### get 流程

1. 计算 `hash`，定位桶，桶为空直接返回 `null`。
2. 首节点匹配（hash 相等且 `equals`）直接返回。
3. 首节点是 `TreeNode` 走红黑树查找，否则顺序遍历链表用 `equals` 匹配。
4. 找不到返回 `null`，注意和「值为 `null`」区分，必要时用 `containsKey` 判断。

### 扩容机制

- 触发：`size > threshold`，或懒初始化、树化前容量不足。
- 结果：容量翻倍，`threshold` 翻倍，老数组元素迁移到新数组。
- JDK 8 迁移优化：不重新计算 `hash`，只看新增的那一位。

```java
// oldCap 为旧容量，判断 hash 的第 oldCap 位
if ((e.hash & oldCap) == 0) {
    // 留在原索引 j，走低位链表 lo
} else {
    // 移到索引 j + oldCap，走高位链表 hi
}
```

- 原理：新容量 `2 * oldCap`，索引只多看一位，该位为 0 索引不变，为 1 索引加 `oldCap`。
- 高低位链表分别保持原相对顺序尾插，不需要逐个重算位置。

### 哈希冲突与退化

- `hashCode` 分布差，或大量 key 人为构造相同 `hashCode`（恶意攻击），桶内链表会变长。
- 链表查找退化为 O(n)，红黑树兜底为 O(log n)，这是 JDK 8 引入树化的核心原因。
- 树化不是万能药：key 必须可比较，否则无法建树，日常仍要保证 `hashCode` 分散。

## 5. 实战应用

### 指定初始容量，避免频繁扩容

```java
// 已知存 100 个元素：(100 / 0.75) + 1 = 135，向上取 2 的幂为 256 也可直接传 135
Map<String, Object> map = new HashMap<>((int) (100 / 0.75f) + 1);
```

- 不指定容量时从 16 开始，存 100 个元素要多次扩容并迁移，得不偿失。
- 经验公式：`初始容量 = (预计元素数 / 0.75) + 1`。

### 遍历时删除

```java
// 错：foreach 中直接 map.remove(k) 会抛 ConcurrentModificationException
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (it.next().getValue() < 0) {
        it.remove(); // 安全删除
    }
}
map.entrySet().removeIf(e -> e.getValue() < 0); // JDK 8 简写
```

### 其他常用写法

- 分组计数：优先 `merge` / `computeIfAbsent`，不要手写判空分支。
- 只读保护：返回内部 map 前用 `Collections.unmodifiableMap(map)` 包一层。
- 有顺序需求：插入顺序用 `LinkedHashMap`，排序顺序用 `TreeMap`，别硬改 `HashMap`。

## 6. 疑难杂症

### 场景 1：重写 equals 忘重写 hashCode，get 不到

- 现象：`put` 进去的对象，用一个 `equals` 相等的新对象 `get` 返回 `null`。
- 原因：默认 `hashCode` 按对象地址算，两个对象 hash 不同，定位到不同桶，根本不会走到 `equals` 比较。
- 修复：`equals` 和 `hashCode` 成对重写，用 IDE 生成或 `Objects.hash(...)`，字段保持一致。

### 场景 2：可变对象作 key，改字段后丢失

- 现象：对象放入后修改了参与 `hashCode` 的字段，再 `get` / `remove` 都失败，数据像凭空消失。
- 原因：放入时按旧 hash 进桶 A，修改后按新 hash 去桶 B 找，永远找不到，原节点还占着内存。
- 修复：key 用不可变对象；必须可变时，先 `remove` 再改字段再 `put`，不要原地改。

### 场景 3：并发下偶发丢数据 / 死循环

- 现象：多线程 `put` 后元素数量对不上，偶发覆盖丢失；JDK 7 还出现过 CPU 100%、扩容成环导致死循环。
- 原因：并发扩容时链表重排互相覆盖；JDK 7 头插法迁移会反转链表顺序，并发下形成环，`get` 时无限循环。
- 现状：JDK 8 改尾插后不再成环，但并发 `put` 数据丢失、覆盖仍存在，不能当线程安全用。
- 修复：并发场景直接换 `ConcurrentHashMap`，或用锁把整个复合操作包住。

## 7. 常见误区

- 误区：`HashMap` 容量传 100，内部就是 100。实际会向上取整为 128。
- 误区：链表长度到 8 就一定树化。容量 < 64 时先扩容，不树化。
- 误区：`hashCode` 相同就认为是同一个 key。还要 `equals` 相等才算。
- 误区：遍历顺序等于插入顺序。`HashMap` 不保证，要顺序用 `LinkedHashMap`。
- 误区：单线程没问题就能挪到多线程。扩容和覆盖的竞态与业务线程数无关。

## 8. 性能优化

- 预估规模并指定初始容量，是收益最大、最简单的一项。
- 优化 key 的 `hashCode`：字段少、分布散，避免用可变字段和劣质算法。
- 负载因子一般不动；内存敏感且查询不频繁时可适当调大，反之调小。
- 热点只读数据：构建完成后不再写的 map 可安全共享读，但仍禁止并发写。
- 大 map 遍历用 `entrySet`，不要先 `keySet` 再逐个 `get` 多查一遍。

## 9. 版本差异

| 对比项 | JDK 7 | JDK 8 |
|---|---|---|
| 数据结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 头节点类型 | `Entry` | `Node` / `TreeNode` |
| 链表插入方式 | 头插法 | 尾插法 |
| 扩容迁移 | 逐个重算索引，头插重排 | 高低位链表拆分，不重算 hash |
| 扩容成环 | 并发扩容可成环、死循环 | 尾插已修成环，仍不保证数据正确 |
| 冲突退化兜底 | 无，最坏 O(n) | 链表 > 8 且容量 ≥ 64 转红黑树 |
| 函数式 API | 无 | `computeIfAbsent`、`merge` 等 |

## 小技巧与冷知识

【小技巧】用 `put` 的返回值判断是否覆盖了旧值，不用先 `get` 再 `put` 查两遍。`put` 会返回该 key 原来的 value，没有旧值时返回 `null`；但 value 本身也可以是 `null`，所以返回 `null` 不能直接断定是新增，必要时先用 `containsKey` 区分。这在“首次放入做初始化、再次放入做更新”的分支里能少一次查找，也少写一次判空。

【小技巧】在 `computeIfAbsent`、`compute`、`merge` 的计算函数里不要再改同一个 map。官方文档明确要求映射函数在计算期间不得修改该 map，否则可能抛异常或得到不一致的结果；需要先把值算好、放到局部变量里，算完再放进 map。原文只讲了这些 API 能省掉 `get` 再 `put` 的两步，容易让人顺手在 lambda 里又 `put` 一次，这正是要避开的坑。

【冷知识】`null` key 永远落在 0 号桶。因为原文 `hash()` 对 `null` 直接返回 0，而索引是 `(n - 1) & hash`，0 与任何数按位与都是 0，所以不管容量多大、扩容多少次，`null` key 都在桶 0。平时只有一个 `null` key 时感觉不到，理解这点后就明白它根本不参与哈希分布。

【冷知识】遍历中途改 map 会抛 `ConcurrentModificationException`，哪怕是单线程。`HashMap` 内部用 `modCount` 记录结构性修改的次数，迭代器创建时记下当时的值，遍历时一旦发现变了就立刻抛错；所以 `foreach` 里直接 `map.remove()` 失败不是多线程问题，单线程也会复现。原文只给了正确写法（用 `Iterator.remove()` 或 `removeIf`），这条解释了它背后的计数机制，看到这个异常先查“遍历时有没有改 map”，而不是先怀疑并发。

## 10. 面试与自测

- `HashMap` 底层结构是什么？为什么引入红黑树？
- 容量为什么必须是 2 的幂？`(n - 1) & hash` 好在哪？
- 树化条件是哪两个？为什么退化阈值是 6 而不是 8？
- 扩容时高低位链表怎么拆？为什么不用重新算 hash？
- 重写 `equals` 为什么必须重写 `hashCode`？
- `HashMap` 为什么线程不安全？JDK 7 和 JDK 8 表现有何不同？
- 已知元素量时初始容量怎么算？
- 自测：手写一个 key 类，故意只重写 `equals`，复现 `get` 返回 `null`，再补上 `hashCode` 验证修复。

### 参考答案

1. 底层是数组 + 链表 + 红黑树。引入红黑树是为兜底哈希冲突退化：链表查找最坏 O(n)，hashCode 分布差或大量 key 被人为构造相同 hashCode 时桶内链表会变长，树化后最坏 O(log n)。
2. 容量 2 的幂：n 为 2 的幂时 (n-1) 低位全是 1，(n-1) & hash 等价于 hash % n，位运算更快且分布均匀。扰动函数 (h = key.hashCode()) ^ (h >>> 16) 把高 16 位异或进低位：容量较小时索引只用低位，高位差异会被浪费，异或后高位也参与定位，分布更均匀；构造时传入非 2 的幂容量会被 tableSizeFor 向上取整到最近的 2 的幂。
3. 树化条件两个同时满足：桶内链表长度 > 8，且容量 ≥ 64；容量 < 64 时先扩容，不树化。退化阈值 6 而不是 8：留出缓冲带，避免在临界点反复树化/退化转换。
4. 扩容不重算 hash：新容量是旧容量 2 倍，索引只多看一位——看 hash 的第 oldCap 位，为 0 留在原索引 j（低位链表），为 1 移到 j + oldCap（高位链表）；两条链表分别保持原相对顺序尾插。
5. 约定：a.equals(b) 为 true 时 a.hashCode() 必须相等；否则两个对象定位到不同桶，根本走不到 equals 比较，get 返回 null。反过来 hashCode 相同、equals 不同只是哈希冲突，桶内再用 equals 精确匹配；且两者要用同一批字段，key 最好不可变。
6. 线程不安全：并发 put/扩容会丢数据、互相覆盖；JDK 7 头插法迁移会反转链表顺序，并发下形成环、get 死循环 CPU 100%；JDK 8 改尾插后不再成环，但并发 put 丢数据、覆盖仍存在，不能当线程安全用，并发场景换 ConcurrentHashMap。
7. 初始容量公式：(预计元素数 / 0.75) + 1，再由 tableSizeFor 向上取整到最近的 2 的幂。按正文示例，存 100 个按公式约 135，实际分配 256；不指定容量从 16 起步，存 100 个要多次扩容迁移。
8. 自测思路：手写 key 类只重写 equals、不重写 hashCode，put 后用 equals 相等的新对象 get，复现返回 null；补上与 equals 同字段的 hashCode 后再验证 get 成功。验证要点：复现时确认两个对象 equals 为 true 但 hashCode 不同、定位到了不同桶；修复后 get/remove 都正常。

## 11. 关联知识

- `ConcurrentHashMap`：并发场景替代品，JDK 8 用 CAS + `synchronized` 锁桶头。
- `LinkedHashMap`：在 `HashMap` 上维护双向链表，保插入/访问顺序，可实现 LRU。
- `TreeMap`：基于红黑树，按 key 排序，复杂度稳定 O(log n)。
- `Hashtable`：老实现，方法级 `synchronized`，不允许 `null` key/value，已很少用。
- Java-并发编程：锁分段、CAS、无锁结构是理解 `ConcurrentHashMap` 的前置知识。
