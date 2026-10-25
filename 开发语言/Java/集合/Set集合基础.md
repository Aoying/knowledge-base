---
知识点: Set集合基础
层级: 基础
适用版本: Java 8+
更新日期: 2026-10-25
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
