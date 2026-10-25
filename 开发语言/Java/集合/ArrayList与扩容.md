---
知识点: ArrayList与扩容
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-23
关联:
  - HashMap
  - LinkedHashMap与LRU
  - CopyOnWriteArrayList
标签:
  - Java
  - 集合
  - 数据结构
---

# ArrayList与扩容

## 1. 核心概念

- `ArrayList` 是基于动态数组的 `List` 实现：底层 `Object[] elementData` 连续存放元素，`size` 记录元素个数。
- 定位：随机访问快、尾部增删快，中间插删要搬移元素。日常默认的 List 选择。
- 非线程安全；允许 `null` 元素；迭代器 fail-fast（见第 4 节）。

| 操作 | ArrayList | LinkedList | 说明 |
|---|---|---|---|
| `get(i)` / `set(i)` | O(1) | O(n) | LinkedList 从近端遍历，平均走 n/4 |
| 尾部 `add` | O(1) 均摊 | O(1) | ArrayList 扩容时偶发 O(n) |
| 头部 / 中间插删 | O(n) 搬移 | O(1)* | *前提是已定位节点；按下标定位仍 O(n)，且每节点多两个引用 |

## 2. 基础知识

```java
List<String> list = new ArrayList<>();
list.add("a");
list.add(0, "b");   // 头部插入，后面元素整体后移
list.get(0);        // "b"
list.remove(0);     // 按下标删
list.remove("a");   // 按对象删，删第一个匹配项
```

- 实现 `RandomAccess` 标记接口：`Collections.binarySearch` 等据此选下标访问路径；`LinkedList` 无此标记，走迭代器路径。
- 构造：`new ArrayList<>(n)` 直接分配 n；`new ArrayList<>(collection)` 拷贝元素。

## 3. 进阶知识：为什么很少用 LinkedList

- `LinkedList` 是双向链表，同时实现 `List` 和 `Deque`，节点含 `item`、`next`、`prev`。
- 按下标访问从近端遍历：`index < size/2` 从头走，否则从尾走，常数减半，量级仍 O(n)。
- 「链表插入 O(1)」的前提是手里已有节点引用；业务代码都是先按下标/值查找再改，查找已是 O(n)，优势不存在；节点内存开销大、缓存不友好，遍历也慢。
- 队列/栈场景也轮不到它：数组实现的 `ArrayDeque` 更快，见 [Queue与Deque](开发语言/Java/集合/Queue与Deque.md)。

## 4. 原理剖析

### 扩容：1.5 倍 + 懒加载

- 无参构造用共享空数组，第一次 `add` 时才扩到默认容量 10。
- 扩容公式：`newCapacity = oldCapacity + (oldCapacity >> 1)`，即 1.5 倍；一次 `addAll` 装不下就直接用所需最小容量。
- 接近上限走 `hugeCapacity`：最大长度 `Integer.MAX_VALUE - 8`。

```java
// grow 简化
int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5 倍
elementData = Arrays.copyOf(elementData, newCapacity);
```

- 为什么不用 2 倍：多次扩容后释放的旧空间有机会被后续复用，内存浪费更低；[HashMap](开发语言/Java/集合/HashMap.md) 用 2 倍是为位运算定位，诉求不同。

### fail-fast

- 结构性修改（add/remove/clear）递增 `modCount`；迭代器创建时记下 `expectedModCount`，`next()` / `remove()` 前校验，不一致抛 `ConcurrentModificationException`。
- 这是尽早暴露 bug 的检测手段，不是同步机制：并发下不能依赖它，并发写用 [CopyOnWriteArrayList](开发语言/Java/集合/CopyOnWriteArrayList.md) 或外部加锁。

### subList 是视图，不是新列表

- `subList(from, to)` 读写直接落到原列表对应区间，不拷贝；原列表在视图之外结构性修改后，再用它抛 `ConcurrentModificationException`。
- 内存坑：subList 持有原列表引用，对大列表取一小段长期保存会拖住整个底层数组；需独立保留时拷贝：`new ArrayList<>(list.subList(0, 10))`。

## 5. 实战应用

```java
// 已知规模时预分配，避免多次扩容拷贝（与 HashMap 指定初始容量同理）
List<Long> ids = new ArrayList<>(rows.size());
// 删全部匹配元素用 removeIf，不要 foreach 里删
list.removeIf(x -> x == null || x.isEmpty());
```

- 互转：`list.toArray(new String[0])`；`Arrays.asList(arr)` 返回定长列表，增删抛 `UnsupportedOperationException`，要可变再包一层 `new ArrayList<>(...)`。

## 6. 疑难杂症

### 场景 1：foreach 里删除抛 ConcurrentModificationException

- 原因：foreach 走迭代器，集合自身 remove 递增 `modCount`，迭代器校验失败。
- 修复：迭代器自己的 `it.remove()`，或 `removeIf`，或收集待删元素后 `removeAll`。

### 场景 2：remove 删错元素

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.remove(1);                  // 删下标 1 的元素（值 2），不是删值 1
list.remove(Integer.valueOf(1)); // 这才是删值 1
```

- `int` 参数优先匹配 `remove(int index)`，装箱类型列表尤其容易踩。

### 场景 3：subList 截取后失效

- 原列表别处 add 一个元素，subList 操作就抛 `ConcurrentModificationException`（共享 modCount 校验）。要快照语义，截取时就拷贝。

## 7. 常见误区

- 误区：默认容量 10 构造时就分配。实际懒加载，首次 add 才扩到 10。
- 误区：fail-fast 能保证并发安全。它只是单迭代器视角的检查。

## 8. 性能优化

- 预估元素量传初始容量，第一优化项；批量添加用 `addAll`，一次扩容到位。
- 对 LinkedList 绝不能用下标 for 遍历，会退化成 O(n²)；用迭代器/for-each。

## 9. 版本差异

- JDK 9+ 新增 `List.of` 不可变工厂，适合小常量列表，但不可变、禁 null，不是 ArrayList 的替代。

## 10. 面试与自测

- 扩容倍数是多少？为什么不是 2 倍？默认容量 10 何时分配？
- fail-fast 是什么？`modCount` 起什么作用？能用来保证线程安全吗？
- subList 有哪些坑？为什么长期持有 subList 可能拖住内存？
- ArrayList 与 LinkedList 怎么选？「链表插入 O(1)」为什么在业务代码里基本兑现不了？
- 自测：复现 foreach 删除的 `ConcurrentModificationException`，再用 `removeIf` 修复。

### 参考答案

1. 扩容 1.5 倍：newCapacity = oldCapacity + (oldCapacity >> 1)，一次 addAll 装不下则直接用所需最小容量，最大长度 Integer.MAX_VALUE - 8。不用 2 倍：多次扩容后释放的旧空间有机会被后续复用，内存浪费更低；HashMap 用 2 倍是为位运算定位，诉求不同。默认容量 10 是懒加载：无参构造用共享空数组，第一次 add 时才扩到 10。
2. fail-fast 是尽早暴露 bug 的检测手段，不是同步机制：结构性修改（add/remove/clear）递增 modCount，迭代器创建时记下 expectedModCount，next()/remove() 前校验，不一致抛 ConcurrentModificationException。它只是单迭代器视角的检查，并发下不能依赖它保证线程安全。
3. subList 是视图不是拷贝：读写直接落到原列表对应区间；原列表在视图之外结构性修改后，再用它就抛 ConcurrentModificationException（共享 modCount 校验）。长期持有 subList 会拖住整个底层数组：subList 持有原列表引用，对大列表取一小段长期保存，原数组无法回收；需独立保留时拷贝：new ArrayList<>(list.subList(0, 10))。
4. 默认选 ArrayList：随机访问 O(1)、尾部增删均摊 O(1)、缓存友好；中间插删频繁且已定位节点时才考虑链表。业务代码里“链表插入 O(1)”兑现不了：前提是手里已有节点引用，业务代码都是先按下标/值查找再改，查找已是 O(n)；且节点内存开销大、缓存不友好，遍历也慢。队列/栈场景用 ArrayDeque 更快。
5. 自测思路：foreach 里直接 list.remove() 复现 ConcurrentModificationException；改用 removeIf（或迭代器 it.remove()、收集待删元素后 removeAll）修复。验证要点：修复后不再抛异常，且删除后剩余元素与预期一致；同时确认这是迭代器校验失败，不是并发问题。

## 11. 关联知识

- [HashMap](开发语言/Java/集合/HashMap.md)：同为非线程安全基础容器，初始容量预估思路相通。
- [LinkedHashMap与LRU](开发语言/Java/集合/LinkedHashMap与LRU.md)：顺序敏感场景的 Map 侧对应物。
- [CopyOnWriteArrayList](开发语言/Java/集合/CopyOnWriteArrayList.md)：读多写极少时的并发 List 方案。
