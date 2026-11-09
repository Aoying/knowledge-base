---
知识点: ArrayList与扩容
层级: 基础
适用版本: JDK 8+
更新日期: 2026-11-09
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

## 10. ArrayList 扩容与复制成本定量：倍数、均摊与浪费算一算

本节只算数：扩容公式、默认容量 10、最大长度的定性结论见第 4 节，这里不再复述。下面所有数字都按“从无参构造出发、逐个 `add`”模拟 `grow()` 得出，复算式见文末附录。

### 换算口径

- 扩容时复制的是底层数组里的引用：每次 `grow()` 把旧数组的全部槽位拷进新数组，复制量 = 旧容量（扩容那一刻数组是满的，`size == oldCapacity`）。
- 内存按引用槽位数换算：64 位 HotSpot 默认开启压缩指针时每个引用 4 字节，未压缩时 8 字节；数组头约 16 字节另计，下面的“约多少 KiB”只算引用槽位。
- “一次扩容”的计数含第一次 0→10（复制 0 个元素）。

### 扩容次数与复制总量：装到 N 要花多少复制

| 装满元素数 N | 扩容次数 | 容量爬升 | 最终容量 | 累计复制元素数 | 均摊复制 / 每次 add | 空位浪费 |
|---|---|---|---|---|---|---|
| 100 | 7 | 10→15→22→33→49→73→109 | 109 | 202 | 2.02 | 9（8.3%） |
| 1000 | 13 | 10→…→823→1234 | 1234 | 2456 | 2.46 | 234（19.0%） |
| 10000 | 19 | 10→…→9369→14053 | 14053 | 28096 | 2.81 | 4053（28.8%） |

- 读法：装 10000 个元素要扩容 19 次，累计复制 28096 个引用——相当于把平均每个元素搬了约 2.8 遍。
- 容量爬升不是精确的 1.5 的幂：`oldCapacity >> 1` 是整数右移，奇数容量会向下取整，例如 15→22（15 + 7），33→49（33 + 16），所以 1000 个元素的最终容量是 1234 而不是 10 × 1.5¹² ≈ 1297。

### 均摊成本：“均摊 O(1)”的数字含义

- 每次扩容复制量 = 旧容量的 100%：扩容那一刻要把已有的全部元素搬一遍，这是最贵的一次操作。
- 但扩容之后有 0.5 倍旧容量的免费 `add`：例如容量从 9369 扩到 14053 后，接下来的 4684 次 `add` 都不用再扩容、零复制。
- 均摊上限：累计复制量 < 3N，即每个 `add` 平均不到 3 次引用复制（3 = 1.5 / (1.5 − 1)）。上表 2.02 / 2.46 / 2.81 正从下方趋近这个上限——N 越大越接近 3，永远不会超过 3。
- 对照：2 倍扩容的均摊上限是 2N / N = 2 次复制（2 = 2 / (2 − 1)），扩容次数更少、总复制更少，但代价在下一节。

### 预估容量 vs 懒扩容：预分配省多少，估大了亏多少

- 已知要装 10000 个元素：`new ArrayList<>(10000)` 一次性分配，0 次扩容、0 次复制，比从 10 开始一路扩容省掉全部 28096 次引用复制和 19 次 `Arrays.copyOf`。
- 峰值内存也省了：懒扩容最后一次扩容的瞬间新旧两个数组同时存在，峰值 9369 + 14053 = 23422 个槽位，约 91.5 KiB（4 字节引用）；预分配峰值就是 10000 个槽位，约 39.1 KiB。
- 反过来，容量估大了的浪费：估 10000 只装 100 个元素，空位 9900 个，约 38.7 KiB（4 字节引用）或约 77.3 KiB（8 字节引用）；估 10000 只装 3000 个元素，空位 7000 个，约 27.3 KiB 或约 54.7 KiB。估大了的代价是线性的：每多估 1000 个空位，多占约 3.9 KiB（4 字节引用）。
- 批量 `addAll` 不走 1.5 倍爬升：已有 10 个元素时一次 `addAll` 10000 个元素，`grow()` 里 `newCapacity < minCapacity` 直接取最小容量，只扩容 1 次到 10010、复制 10 个元素——批量装数据比逐个 `add` 省 28086 次复制。

### 1.5 倍 vs 2 倍：内存浪费对比

| 装满元素数 N | 1.5 倍最终容量 / 浪费 | 1.5 倍累计复制 | 2 倍最终容量 / 浪费 | 2 倍累计复制 |
|---|---|---|---|---|
| 100 | 109 / 9（8.3%） | 202 | 160 / 60（37.5%） | 150 |
| 1000 | 1234 / 234（19.0%） | 2456 | 1280 / 280（21.9%） | 1270 |
| 10000 | 14053 / 4053（28.8%） | 28096 | 10240 / 240（2.3%） | 10230 |

- 具体 N 的浪费要看落点：10000 恰好贴在 2 的幂附近，2 倍这一次反而只浪费 240 个槽位（2.3%），比 1.5 倍的 4053 个更省——单点对比不能说明倍数优劣。
- 平均意义上 1.5 倍更省：按容量落点在对数尺度上均匀分布估算（经验口径，非官方数字），期望容量 / 元素数 1.5 倍约 1.23（平均浪费约 23%），2 倍约 1.44（平均浪费约 44%）。
- 旧空间复用的数字版（经验口径，非 JDK 官方结论；第 1 问的结论只说“旧空间有机会被后续复用”，这里补数字）：装到 10000 时，1.5 倍一路释放的旧块合计 28096 个槽位，而下一次扩容新数组只要 21079 个槽位（14053 + 7026），历史上释放的空间总量已超过下一次需求；2 倍释放合计 10230 个槽位，下一次需求 20480 个，不到需求的一半。倍数小于 2 时，释放量的累积有可能盖住下一次分配；倍数等于 2 时，释放量的累积永远盖不住。

### remove / 中间插入的搬移成本

- 单次搬移量只看下标，不看倍数：`remove(i)` 搬 `size − i − 1` 个，`add(i, e)` 搬 `size − i` 个。
- 10000 个元素的列表：删下标 5000 的元素搬 4999 个；下标 5000 处插入搬 5000 个；删下标 0 搬 9999 个；删末尾搬 0 个。
- 批量删除的差距：从下标 5000 起逐个 `remove` 掉 5000 个元素，累计搬移 12497500 个，约 1250 万；`removeIf` 一次遍历，被保留的 5000 个元素每个最多搬一次，总搬移 ≤ 5000 个——相差约 2500 倍。从下标 0 起逐个删 5000 个更贵，累计搬移 37497500 个，约 3750 万。

### 边界条件

- 换算假设：元素为引用类型，底层数组存的是引用而非对象本身；4 字节按 64 位 HotSpot 默认压缩指针、8 字节按未压缩，引用宽度随 JVM 与 `-XX` 配置而变，字节数按比例换算；复制量按“扩容瞬间数组已满”计算，`addAll` 一次装入大量元素时不适用。
- `minCapacity` 兜底：`grow()` 里算出的 1.5 倍容量小于所需最小容量时直接取最小容量；一次 `addAll` 装不下不会按 1.5 倍多次爬升，而是 1 次扩容到位。
- `hugeCapacity` 上限：`MAX_ARRAY_SIZE = Integer.MAX_VALUE − 8 = 2147483639`；1.5 倍算出的容量超过它时走 `hugeCapacity`，所需容量更大时取 `Integer.MAX_VALUE`，所需容量溢出为负数时抛 `OutOfMemoryError`。
- `subList` 视图不参与扩容：它不建新数组、不触发 `grow()`，结构性修改走原列表（见第 4 节）。
- 版本口径：1.5 倍的量级自 JDK 6 起未变，但精确公式在 JDK 7 有过调整——JDK 6 是 `(oldCapacity * 3) / 2 + 1`，JDK 7 起改为 `oldCapacity + (oldCapacity >> 1)`（去掉 `+1`）；JDK 8 的懒加载、JDK 9+ 把 `grow(minCapacity)` 重构为 `newCapacity(minCapacity)`，公式与上限均无变化；本节数字按 JDK 8+ 源码复算，与原文适用版本 JDK 8+ 一致。

## 小技巧与冷知识

【小技巧】列表装完就定型时，调一次 `trimToSize()` 把多余容量还回去：预分配 10000 只装了 3000 个元素，底层数组剩下的 7000 个空位会一直占着内存。`trimToSize()` 把数组缩到刚好等于 size，适合“一次性加载、长期持有、基本不再写”的列表；但缩完再 `add` 会立刻重新扩容，写操作还频繁的列表别用它。

【小技巧】容量在运行时才知道时，用 `ensureCapacity(n)` 预先扩容：构造时传容量只适合编译期就知道规模的场景，容量是查库行数、文件行数这种运行时才有的数字时，拿到数字后先调 `ensureCapacity(n)` 再循环 `add`。效果和构造时传容量一样，都是避免循环里反复扩容拷贝；批量装数据前先问一句“这次大概装多少”，是 ArrayList 最划算的优化。

【小技巧】`clear()` 只清元素不缩容：它把 size 置 0、把元素引用置 null 方便 GC，但底层数组的容量原样保留。清空后长期持有这个列表，容量还是扩容后的峰值；想把容量也还回去，要么 `trimToSize()`，要么直接新建一个列表。复用一个“峰值很大、平时很空”的列表时，这点内存要算进去。

【小技巧】大列表扩容的瞬间内存占两份：`Arrays.copyOf` 是先建好新数组、再把旧元素拷过去，旧数组要等拷贝完成才释放。一个接近堆上限的大列表扩容时，同一时刻同时存在新旧两个数组，峰值内存接近两倍容量，恰好在临界点可能触发 OOM。已知规模时预分配不只是省拷贝次数，也是把内存峰值从“两倍”压回“一倍”。

【小技巧】批量删元素用 `removeIf`，别在循环里逐个 `remove`：每次 `remove(i)` 都要把后面的元素整体前移一次，删 k 个元素就是 k 次搬移；`removeIf` 内部先用位图标记要删的位置，再一次遍历把保留的元素紧凑排好，只搬移一次。删除量越大差距越明显，删一半元素时循环 remove 是 O(n²)，`removeIf` 是 O(n)。

【小技巧】`removeAll` 的参数传 `HashSet` 别传 `List`：`removeAll` 内部对每个元素调一次参数集合的 `contains`，参数是 List 时 `contains` 本身是线性扫描，整体退化成 O(n·m)。待删集合先装进 `HashSet` 再传进去，`contains` 变成 O(1)，整体回到 O(n)；待删集合本来就是 List 时，包一层 `new HashSet<>(toRemove)` 的成本远小于逐个扫描。

【小技巧】`remove(Object)` 删不掉“看起来一样”的对象时，先查 `equals`：ArrayList 按值查找和删除都靠元素的 `equals` 方法，自定义类没重写 `equals` 时用的是 `Object` 的身份比较，两个字段完全相同的对象也会被当成不同对象。删之前先确认这个类有没有按业务字段重写 `equals`（以及配套的 `hashCode`），没重写就先补上，否则删、查、去重全都会按引用走。

【小技巧】现代 JDK 里 `toArray` 传 `new String[0]` 是更推荐的写法：老写法 `toArray(new String[size])` 是为了在旧版本上省一次反射建数组，JDK 11 之后 JVM 对零长度数组的 `toArray` 有专门优化，会按实际大小一次分配到位，传预 sized 数组反而多一次初始化。写法上 `list.toArray(new String[0])` 也更短，容量变化时不用再改参数。

【小技巧】按值查找是线性扫描，别拿 ArrayList 当索引用：`indexOf`、`contains`、`remove(Object)` 都是从头扫到尾，元素一多就慢。业务里需要“按某个字段反复查、反复判重”时，先建一个 `HashSet` 或 `Map` 当索引，列表只负责保序；列表和索引各干各的，查和删都回到 O(1)，代价是写操作要同步维护两份结构。

【冷知识】无参构造和 `new ArrayList<>(0)` 用的不是同一个空数组：前者用 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，第一次 `add` 直接扩到默认容量 10；后者用 `EMPTY_ELEMENTDATA`，第一次 `add` 只扩到 1。`new ArrayList<>(0)` 的语义是“我明确知道现在一个元素都没有”，扩容从 1 开始按 1.5 倍爬；两个空数组长得一样，第一次扩容的落点差了 10 倍。

【冷知识】`hasNext()` 不做 fail-fast 校验，改完集合后它还能返回 true：迭代器的 `modCount` 检查只在 `next()` 和 `remove()` 里做，`hasNext()` 只比较下标和 size。循环里改了集合再调 `hasNext()`，它照常返回 true，真正抛 `ConcurrentModificationException` 的是随后那次 `next()` 调用；排障时别被“hasNext 还正常”误导，异常点永远在 `next()` 那一行。

【冷知识】`Arrays.asList` 返回的不是 `java.util.ArrayList`：它是 `Arrays` 的私有内部类 `Arrays$ArrayList`，和 `java.util.ArrayList` 没有继承关系，只是都实现了 `List`。它底层直接包着传入的数组，改它就是改原数组，增删抛异常也是因为它根本没有扩容机制——名字里的 ArrayList 是巧合，不是同一个类。

【冷知识】`elementData` 是 `transient` 的，序列化时不写整个数组：ArrayList 自定义了 `writeObject`，只写实际元素个数和这几个元素，数组里多余的空位不进序列化流。一个容量 10000、只装了 10 个元素的列表序列化后只有 10 个元素的数据量，不会把 9990 个空位也写进去；反序列化时再按元素个数重建数组，容量不会原样保留。

## 11. 面试与自测

- 扩容倍数是多少？为什么不是 2 倍？默认容量 10 何时分配？
- fail-fast 是什么？`modCount` 起什么作用？能用来保证线程安全吗？
- subList 有哪些坑？为什么长期持有 subList 可能拖住内存？
- ArrayList 与 LinkedList 怎么选？「链表插入 O(1)」为什么在业务代码里基本兑现不了？
- 自测：复现 foreach 删除的 `ConcurrentModificationException`，再用 `removeIf` 修复。

- 从默认容量 10 起逐个 `add` 装满 10000 个元素，要扩容几次、累计复制多少个元素？预分配 `new ArrayList<>(10000)` 能省多少？

### 参考答案

1. 扩容 1.5 倍：newCapacity = oldCapacity + (oldCapacity >> 1)，一次 addAll 装不下则直接用所需最小容量，最大长度 Integer.MAX_VALUE - 8。不用 2 倍：多次扩容后释放的旧空间有机会被后续复用，内存浪费更低；HashMap 用 2 倍是为位运算定位，诉求不同。默认容量 10 是懒加载：无参构造用共享空数组，第一次 add 时才扩到 10。
2. fail-fast 是尽早暴露 bug 的检测手段，不是同步机制：结构性修改（add/remove/clear）递增 modCount，迭代器创建时记下 expectedModCount，next()/remove() 前校验，不一致抛 ConcurrentModificationException。它只是单迭代器视角的检查，并发下不能依赖它保证线程安全。
3. subList 是视图不是拷贝：读写直接落到原列表对应区间；原列表在视图之外结构性修改后，再用它就抛 ConcurrentModificationException（共享 modCount 校验）。长期持有 subList 会拖住整个底层数组：subList 持有原列表引用，对大列表取一小段长期保存，原数组无法回收；需独立保留时拷贝：new ArrayList<>(list.subList(0, 10))。
4. 默认选 ArrayList：随机访问 O(1)、尾部增删均摊 O(1)、缓存友好；中间插删频繁且已定位节点时才考虑链表。业务代码里“链表插入 O(1)”兑现不了：前提是手里已有节点引用，业务代码都是先按下标/值查找再改，查找已是 O(n)；且节点内存开销大、缓存不友好，遍历也慢。队列/栈场景用 ArrayDeque 更快。
5. 自测思路：foreach 里直接 list.remove() 复现 ConcurrentModificationException；改用 removeIf（或迭代器 it.remove()、收集待删元素后 removeAll）修复。验证要点：修复后不再抛异常，且删除后剩余元素与预期一致；同时确认这是迭代器校验失败，不是并发问题。

6. 从 10 起按 1.5 倍扩容，装满 10000 个元素要扩容 19 次，最终容量 14053，累计复制 28096 个引用，均摊约 2.8 次/add。预分配 new ArrayList<>(10000) 则零扩容、零复制，省掉 28096 次复制；代价是容量一次给满、空位占内存。

## 12. 关联知识

- [HashMap](开发语言/Java/集合/HashMap.md)：同为非线程安全基础容器，初始容量预估思路相通。
- [LinkedHashMap与LRU](开发语言/Java/集合/LinkedHashMap与LRU.md)：顺序敏感场景的 Map 侧对应物。
- [CopyOnWriteArrayList](开发语言/Java/集合/CopyOnWriteArrayList.md)：读多写极少时的并发 List 方案。
