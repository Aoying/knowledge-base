---
知识点: Set集合基础
层级: 基础
适用版本: Java 8+
更新日期: 2026-10-31
关联:
  - HashMap
  - TreeMap与红黑树
  - LinkedHashMap与LRU
标签:
  - Java
  - 集合
  - 实践
---

# Set集合基础

## 1. 核心概念

- `Set` 是“不重复元素的集合”：语义只有去重，没有索引，不能按位置取元素。
- 去重不等于有序：`HashSet` 的遍历顺序不固定；要插入顺序用 `LinkedHashSet`，要排序用 `TreeSet`，顺序需求必须显式选实现。
- `Set` 的去重标准由实现决定：哈希实现看 `equals/hashCode`，排序实现看比较器返回是否为 0，两套标准不要混用。
- 典型用途：判重（黑名单、已处理 ID）、去重（日志 ID、标签）、集合运算（交集、并集、差集）。

## 2. 基础知识

### 2.1 四种实现一览

| 实现 | 底层结构 | 遍历顺序 | 复杂度 | null 元素 |
|---|---|---|---|---|
| `HashSet` | `HashMap`（value 是固定占位对象） | 不固定 | 增删查平均 O(1) | 允许 1 个 |
| `LinkedHashSet` | 哈希表 + 双向链表 | 插入顺序 | 增删查平均 O(1) | 允许 1 个 |
| `TreeSet` | 红黑树 | 按比较器排序 | 增删查 O(log n) | 不允许（无比较器时抛异常） |
| `EnumSet` | 位向量（`long` 或 `long[]`） | 枚举声明顺序 | 增删查接近 O(1) | 不允许 |

- `CopyOnWriteArraySet` 是 `CopyOnWriteArrayList` 的 Set 包装，读多写极少且需线程安全时用；写时全量复制，写多会很慢。
- Java 9 起 `Set.of(...)` 可创建不可变 Set：元素不允许为 `null`，重复元素直接抛 `IllegalArgumentException`，适合常量集合。
- Java 21 起 `LinkedHashSet` 实现了 `SequencedSet`，可用 `getFirst/getLast/reversed()` 等统一的有序操作（Java 21）。

### 2.2 常用 API

- 增删查：`add`（已存在返回 `false`，这是判重的常用写法）、`remove`、`contains`。
- 集合运算：`addAll`（并集）、`retainAll`（交集）、`removeAll`（差集），都会原地修改当前集合；要保留原集合先 `new HashSet<>(原集合)` 拷贝一份。
- 遍历：增强 for、`forEach`、迭代器；`Set` 没有 `get(int)`，想按位置取先转 `List`。

## 3. 原理剖析

### 3.1 哈希约定：HashSet/LinkedHashSet 的地基

- 只有一条铁律：`equals` 相等的对象，`hashCode` 必须相等；反过来不要求（冲突允许，用 `equals` 二次确认）。
- 自定义元素必须同时重写两个方法：只重写 `equals` 会导致“逻辑上相等”的两个对象哈希不同，`HashSet` 认不出来，`contains` 返回 `false`。
- `hashCode` 必须用参与 `equals` 比较的字段计算；拿无关字段算哈希，等于没实现约定。

### 3.2 排序约定：TreeSet 的地基

- `TreeSet` 不看 `hashCode`，只看比较器：构造时传 `Comparator`，或元素实现 `Comparable`（自然排序）。
- 比较器返回 0 即视为相等：`TreeSet.add` 会拒绝加入，哪怕两个对象的 `equals` 并不相等。比较逻辑与相等逻辑不一致时，去重行为会出乎意料。
- 比较器必须满足自反、反对称、传递三性；比较逻辑里混用多个字段时注意溢出（整数相减可能溢出，优先用 `Integer.compare`）。

### 3.3 EnumSet 的位向量

- `EnumSet` 不用哈希也不用红黑树：枚举常量个数固定，直接用位向量表示“在不在集合里”，一个 `long` 可表示 64 个枚举值。
- 因此它比 `HashSet` 更省内存、更快，但只能装同一枚举类型的常量，且不允许 `null`。
- 创建用静态工厂：`EnumSet.of(...)`、`EnumSet.allOf(...)`、`EnumSet.noneOf(...)`，没有公开构造器。

## 4. 选型指南

| 场景 | 首选 | 理由 |
|---|---|---|
| 只判重/去重，不关心顺序 | `HashSet` | 平均 O(1)，最省事 |
| 去重且要保留插入顺序 | `LinkedHashSet` | 哈希 + 链表，顺序稳定 |
| 去重要排序（如排行榜、区间） | `TreeSet` | 天然有序，支持范围操作（`subSet/headSet/tailSet`） |
| 枚举类型的小集合 | `EnumSet` | 位向量，最快最省 |
| 常量集合 | `Set.of`（Java 9） | 不可变，防误改 |
| 读多写极少 + 多线程 | `CopyOnWriteArraySet` | 无锁读，写时复制 |

## 5. 实战应用与常见坑

- 去重实战：`new HashSet<>(list)` 一行去重；要保序用 `new LinkedHashSet<>(list)`；去重后转回 `List` 用 `new ArrayList<>(set)`。
- 判重写法：`if (!seen.add(id))` 表示“已处理过”，比先 `contains` 再 `add` 少一次查找。
- 可变对象作元素是大坑：对象进 `HashSet` 后改了参与 `hashCode` 的字段，哈希桶位置变了，`contains/remove` 就找不到了，集合里出现“删不掉的脏数据”。规矩：进 Set 的对象，参与哈希/比较的字段必须不可变。
- `TreeSet` 比较与相等不一致坑：`compareTo` 只比了年龄、`equals` 比了全部字段时，两个同年龄不同人会被 `TreeSet` 当成同一个而去重。修法是比较逻辑与相等逻辑用同一组字段。
- `Set.of` 的两个雷：元素重复直接抛异常（不是静默去重）、元素为 `null` 抛 `NullPointerException`，构造常量集合前先保证输入干净。
- 并发坑：普通 `Set` 多线程读写必须换并发实现或外部加锁；`Collections.synchronizedSet` 只保证单操作安全，遍历仍需手动 `synchronized (set)` 包住整个循环。

## 小技巧与冷知识

【小技巧】`TreeSet` 的 `subSet/headSet/tailSet` 返回的是“活视图”，不是拷贝：对子集视图增删元素会直接改原集合，反过来原集合在视图范围内的改动也会在视图里看见。拿子集做区间统计、批量删除时很顺手，但别把它当快照用——想定格一份区间数据，先 `new TreeSet<>(set.subSet(...))` 拷一份；遍历视图时原集合在范围外改动同样可能触发 fail-fast。

【小技巧】想倒序遍历 `TreeSet` 别先拷成 List 再倒排，直接用 `descendingSet()` 或 `descendingIterator()`：`descendingSet()` 返回的仍是活视图，倒序遍历零拷贝；`descendingIterator()` 一次倒序走完就够。排行榜“取后 N 名”这类需求，`set.descendingSet().stream().limit(n)` 一行搞定，比先转 List 再 `Collections.reverse` 少一次全量拷贝。

【小技巧】`LinkedHashSet` 里重复 `add` 一个已存在的元素，不会把它挪到队尾：它是插入顺序，不是访问顺序——已存在的元素 `add` 返回 `false`、位置纹丝不动。想“最近用过的排最后”得用访问顺序的 `LinkedHashMap`，或者先 `remove` 再 `add`；指望重复 `add` 刷新顺序，淘汰逻辑会悄悄算错。

【小技巧】遍历中删元素用迭代器的 `remove()` 或 `removeIf`，别在增强 for 里调 `set.remove()`：增强 for 内部就是迭代器，循环体里直接删会改 `modCount`、下一次 `hasNext` 抛 `ConcurrentModificationException`。批量按条件删直接用 `removeIf(条件)`，它内部一次遍历完成，比手写迭代器循环更短；想“删一批、留一批”时 `retainAll` 同样是一次原地操作。

【小技巧】元素类型优先用 record 或不可变类，别拿可变 POJO 当 Set 元素：record 会按全部组件自动生成一致的 `equals/hashCode`，字段全是 final，天然满足“参与哈希的字段不可变”这条规矩。手写 POJO 时 `equals/hashCode` 容易只写一半字段、或漏掉一个字段；用 record 或 Lombok 的 `@Value`，去重语义由编译器兜底，而不是靠每次 code review 人肉检查。

【小技巧】`Set.of` 和 `Collections.unmodifiableSet` 不是一回事：`Set.of` 是彻底不可变的独立拷贝，元素不能为 `null`、不能重复；`unmodifiableSet` 只是给原集合包一层“只读壳”，原集合改了，透过壳读到的内容也会变。常量集合用 `Set.of`，想“对外只读、对内可改”才用 `unmodifiableSet`；把后者当常量用，内部一改就从外部漏出去。

【小技巧】`EnumSet` 还有 `range` 和 `complementOf` 两个少用的工厂：`EnumSet.range(周一, 周五)` 一行取连续一段枚举，`EnumSet.complementOf(工作日)` 直接取补集，位运算在内部完成。权限位、星期、状态机这种“枚举子集”场景，别手写循环一个个 `add`，工厂方法本身就是按位向量设计的入口。

【小技巧】`CopyOnWriteArraySet` 的 `add` 是 O(n)，别拿它当高频判重表：它的 `add` 内部先线性扫描整个数组确认不存在再拷贝写入，元素一多、每次 `add` 都是一次全表扫描加一次全表拷贝。读多写极少的小监听器集合才适合它；判重频率高、数据量上千时，`ConcurrentHashMap.newKeySet()` 才是 O(1) 的选择。

【小技巧】流式去重想保序用 `distinct()`，别先 `collect(toSet())` 再排序：有序流的 `distinct()` 会保留元素首次出现的顺序，内部按插入顺序去重，效果等同 `new LinkedHashSet<>(list)`。无序流或并行流的 `distinct()` 不保证顺序；顺序有要求时先确认流是有序的，或者显式用 `LinkedHashSet` 收尾，别把顺序寄托在 `HashSet` 的遍历上。

【小技巧】`HashSet` 有初始容量构造器，大集合去重前先估规模：注意 `new HashSet<>(n)` 里的 n 是初始桶容量，不是“期望元素数”——扩容阈值是容量 × 加载因子 0.75，传 10000 只装得下 7500 个元素不扩容；要装 N 个元素不触发扩容，应传 `(int)(N / 0.75f) + 1`。已知要装 10 万个 ID 时按这个公式给够容量，避免装到一半反复扩容重哈希；估不准也别乱给一个极大值，桶数组本身占内存，容量和元素量要成比例。

【小技巧】`TreeSet` 里 `null` 能不能放，看比较器而不是看 Set：无比较器时 `add(null)` 直接抛 `NullPointerException`，因为自然排序没法比较 `null`；传了 `Comparator.nullsFirst(...)` 或 `nullsLast(...)` 的比较器，`null` 就能正常入队。判空逻辑别写在 `add` 之前，先确认比较器对 `null` 的态度，`null` 在 `TreeSet` 里是“比较器说了算”，不是“Set 说了算”。

【小技巧】需要“元素个数”而不是“去重”时，别用 Set 充数：Set 会静默丢掉重复元素，统计词频、计数出现次数时用它等于主动丢数据。计次用 `Map<元素, Integer>` 或 `merge(key, 1, Integer::sum)`，多重集语义用 Guava 的 `Multiset`；Set 只回答“在不在”，不回答“有几个”，拿它计数是语义错配。

【冷知识】`Set.of` 的遍历顺序不保证，而且同一份输入在不同次运行里顺序可能不同：JDK 文档只说迭代顺序未指定，实现里用了每次启动随机的盐值打乱顺序。所以别拿 `Set.of(a, b, c)` 的打印顺序写断言，也别指望它和 `HashSet` 的顺序一致——不可变集合的顺序是“未定义”，不是“插入顺序”也不是“哈希顺序”，测试里按顺序比较 `Set.of` 的元素是注定会 flaky 的写法。

【冷知识】`Set` 接口是 JDK 1.2（1998 年）随集合框架一起进 JDK 的：Java 1.0/1.1 时代只有 `Vector`、`Hashtable` 这些遗留类，没有统一的集合接口；JDK 1.2 由 Joshua Bloch 主导的集合框架一次性带来了 `Collection`、`Set`、`List`、`Map` 和 `HashSet`、`TreeSet` 等实现。`EnumSet`、`CopyOnWriteArraySet`、`Set.of` 都是后来才补上的——Set 的核心语义二十多年没变，变的是实现和构造方式。

【冷知识】`HashSet` 只是 `HashMap` 的一层薄壳，`TreeSet` 也只是 `TreeMap` 的薄壳：`HashSet` 内部就是一个 `HashMap`，元素当 key、value 是同一个固定的占位对象；`TreeSet` 内部就是一个 `TreeMap`，排序、范围操作全是 `NavigableMap` 的能力透传。这也是为什么 `HashSet` 的遍历顺序、扩容行为和 `HashMap` 一模一样——理解了 `HashMap`，`HashSet` 就只剩“value 被藏起来了”这一个差别。

## 6. 面试与自测

- `HashSet`、`LinkedHashSet`、`TreeSet` 各自的遍历顺序是什么？分别由什么机制保证？
- 为什么重写 `equals` 必须重写 `hashCode`？只重写 `equals` 时 `HashSet` 会出什么具体问题？
- `TreeSet` 是靠 `equals` 去重的吗？比较器返回 0 但 `equals` 为 `false` 的两个对象会怎样？
- `EnumSet` 为什么比 `HashSet` 快？它有什么使用限制？
- 自测：写一个可变 `Person` 类（`name` 参与 `equals/hashCode`），先 `add` 进 `HashSet` 再改 `name`，验证 `contains` 与 `remove` 是否还能找到它，并说清原因。

### 参考答案

1. `HashSet` 遍历顺序不固定，由哈希桶分布决定，扩容后可能变化；`LinkedHashSet` 是插入顺序，靠哈希表之外的双向链表维护；`TreeSet` 是比较器定出的排序，由红黑树的中序遍历保证。顺序需求必须显式选实现，不能默认 `HashSet` 有序。
2. 因为哈希容器先靠 `hashCode` 定位桶、再靠 `equals` 精确匹配。只重写 `equals` 时，两个逻辑相等的对象哈希值不同，会落到不同桶，`HashSet` 认不出它们相等：`add` 会重复收录、`contains` 返回 `false`，去重彻底失效。铁律是 `equals` 相等则 `hashCode` 必相等，且 `hashCode` 必须用参与 `equals` 的字段计算。
3. 不是。`TreeSet` 只看比较器：比较器返回 0 即视为相等，`add` 会拒绝加入第二个对象，哪怕它们的 `equals` 并不相等。后果是“比较逻辑定义了去重标准”，比较字段少于相等字段时会出现误去重，修法是比较逻辑与相等逻辑用同一组字段。
4. `EnumSet` 用位向量表示集合：枚举常量个数固定，一个 `long` 可表示 64 个值，增删查就是位运算，没有哈希计算也没有树操作，所以更快更省。限制是只能装同一枚举类型的常量，不允许 `null`，且只能通过 `EnumSet.of/allOf/noneOf` 等静态工厂创建。
5. 验证结果：改名后 `contains` 与 `remove` 都找不到该对象。原因是对象进集合时按旧 `name` 的哈希值落桶，改名后哈希值变了，查找按新哈希值去另一个桶找，自然找不到；对象实际还在旧桶里，成了删不掉的脏数据。教训是进 Set 的对象中参与哈希/比较的字段必须不可变。

## 7. 关联知识

- `集合框架总览`：全景与选型，本篇是其中 Set 一节的展开。
- `HashMap`：`HashSet` 底层就是 `HashMap`，哈希约定与扩容细节以该篇为准。
- `TreeMap与红黑树`：`TreeSet` 底层是 `TreeMap`，红黑树机制以该篇为准。
- `LinkedHashMap与LRU`：`LinkedHashSet` 的链表保序思想与 `LinkedHashMap` 同源。
