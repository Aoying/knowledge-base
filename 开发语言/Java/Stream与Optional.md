---
知识点: Stream 与 Optional
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java
标签:
  - Java
  - 实践
---

# Stream 与 Optional

## 1. Stream：集合管道

Stream 是对集合的声明式处理管道：你描述「要什么」，不写「怎么遍历」。一条管道由三部分组成：数据源（集合/数组/IO）、中间操作（`filter`、`map`、`sorted`、`limit` 等，惰性求值，只登记不执行）、终端操作（`collect`、`forEach`、`reduce`、`count` 等，触发整条管道真正跑起来）。理解惰性求值是理解 Stream 行为的关键：没有终端操作，中间操作一行都不执行。

```java
// Stream：集合管道，一行顶五行 for
List<String> names = users.stream()
    .filter(u -> u.getAge() >= 18)
    .sorted(Comparator.comparing(User::getScore).reversed())
    .limit(10)
    .map(User::getName)
    .collect(Collectors.toList());
```

常用操作速记：

- `filter` 过滤；`map` 一对一转换；`flatMap` 一对多再拉平（拆嵌套集合、把 `Stream<Optional<T>>` 摊平都靠它）。
- `distinct` 去重（依赖 equals/hashCode）；`sorted` 排序；`peek` 只用于调试打印，别在里面改状态。
- 终端：`collect(Collectors.toList()/toMap()/groupingBy())` 分组聚合是业务里最常用的；`reduce` 做累计计算；`anyMatch/allMatch/noneMatch` 做判断；`findFirst/findAny` 返回 Optional。
- 基本类型流 `IntStream/LongStream/DoubleStream` 避免装箱，`IntStream.range(0, n)` 替代带下标的 for 很顺手。

分组与收集的例子（业务高频）：

```java
// 按状态分组统计金额
Map<Integer, BigDecimal> sumByStatus = orders.stream().collect(
    Collectors.groupingBy(Order::getStatus,
        Collectors.reducing(BigDecimal.ZERO, Order::getAmount, BigDecimal::add)));

// toMap 必须处理 key 冲突，否则重复 key 直接抛异常
Map<Long, User> byId = users.stream().collect(
    Collectors.toMap(User::getId, u -> u, (a, b) -> a)); // 冲突保留先出现的
```

## 2. 并行流的取舍

`parallelStream()` 把任务拆给 ForkJoinPool 公共池跑。听起来免费提速，实际只在「数据量大、单元素处理重、且无共享状态」时才划算；普通业务集合（几百几千条、元素处理就是简单取值判断）并行化的拆分调度开销反而更慢（经验）。更危险的是它共用全局公共池，在里面做阻塞 IO 会拖累同 JVM 里其他并行流。结论：默认不用，确认是 CPU 密集大批量再考虑，并实测对比。

## 3. Optional：把「可能没有」写进类型

`Optional` 的设计意图是做**方法返回值**：调用方从签名上就知道结果可能为空，必须显式处理，而不是靠文档和自觉判空。

```java
// Optional：告别 != null 金字塔
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("未知");
```

- 创建：`Optional.of(x)`（x 为 null 直接抛异常，用于确非空）、`ofNullable(x)`（可能空）、`Optional.empty()`。
- 取值：`orElse(默认值)` 参数会**总是先求值**，默认值构造贵时用 `orElseGet(() -> ...)` 惰性构造；没有就该报错时用 `orElseThrow(() -> new BizException("..."))`，别调 `get()`——`get()` 在空时抛 `NoSuchElementException`，和 NPE 一样难定位。
- 组合：`map` 里别再 `return null`，用 `flatMap` + `Optional` 保持链式（返回 Optional 的 getter 用 flatMap 接）；`filter` 可在链上加条件；`ifPresentOrElse`（JDK 9+）处理「有则消费、无则兜底」。
- JDK 版本差异：`stream()`（把 Optional 转成 0/1 元素流）是 JDK 9+ 才有，JDK 8 项目别写。

## 4. 常见误区

- **Stream 复用**：Stream 是"用一次就废"，复用同一个 stream 会抛 `IllegalStateException`（已被操作或关闭）。需要重复处理就每次从集合重新 `.stream()`。
- **拿 Optional 当字段和入参**：实体字段、DTO 入参不要用 Optional 包——序列化框架支持参差、语义也错（入参缺失与值为 null 是两回事）；Optional 只做返回值。
- **Optional 判空写回老样子**：`if (opt.isPresent()) { x = opt.get(); }` 等于白用；用 `map/orElse` 链或 `ifPresent` 消费。
- **在 Stream 里改外部状态**：`forEach` 里往外部 List 里 add 是并发不安全且语义不清的，收集结果一律走 `collect`。
- **排序稳定性**：对已按某字段排序的流再 `sorted` 其他字段，Stream 对有序流保证稳定排序，List 流就是有序的，这点可以依赖。
- 性能敏感的大循环还是老 for，Stream 的可读性优先于那点性能——前提是先确认它真是热点，别凭感觉在业务代码里到处回退 for。


## 小技巧与冷知识

【小技巧】`IntStream.range(0, n)` 的结束是开区间，别写成 `rangeClosed` 的语义：`range(0, 3)` 只出 0、1、2，`rangeClosed(0, 3)` 才出 0、1、2、3。原文只给了 `range(0, n)` 替代下标 for 的例子，换下标循环时先想清楚原来 for 的结束条件是 `< n` 还是 `<= n`，写错一个方法名就是差一错误，编译器不会拦。

【小技巧】`limit` 要放在 `sorted` 之前还是之后，结果完全不同：`sorted().limit(10)` 是“先排好序再取前 10”，`limit(10).sorted()` 是“先取前 10 个再排这 10 个”，后者取的是原顺序的前 10 条、不是最大的 10 条。短路操作放的位置决定了它截断的是哪一步的结果，链式顺序调换前先在纸上走一遍数据。

【小技巧】`Collectors.toList()` 不保证返回哪种 List：JDK 文档明确写了不保证返回列表的类型、可变性与线程安全性，JDK 8 到 15 的实现返回的是 ArrayList、可以改，但别把这个实现细节当契约用。想要确定不可变的列表，用 JDK 16 起的 `Stream.toList()`；想要确定可变的，先 `collect` 再 `new ArrayList<>(...)` 包一层。

【小技巧】`reduce` 的 identity 必须是“中性元”：求和用 `0`、求积用 `1`、拼字符串用 `""`，identity 错了并行流下结果直接错。`reduce` 不带 identity 的重载返回的是 `Optional`，空流时是 `Optional.empty()` 而不是抛异常，适合“空了就该没有结果”的聚合；原文的 `groupingBy` + `reducing` 例子用的是带 identity 的版本，两种重载按“空流怎么办”选。

【小技巧】`flatMap(Optional::stream)` 是 JDK 9 起摊平 `Stream<Optional<T>>` 最干净的写法：`Optional.stream()` 把有值转成 1 个元素的流、空转成空流，`flatMap` 一接就只剩有值的元素。原文 §1 只说“把 `Stream<Optional<T>>` 摊平靠 flatMap”，这行就是具体的写法；JDK 8 项目没有 `Optional.stream()`，只能 `filter(Optional::isPresent).map(Optional::get)` 绕过去。

【小技巧】`Optional.or()`（JDK 9+）是“有则用、无则换另一个 Optional”的链式兜底：`opt.or(() -> loadFromCache()).or(() -> loadFromDb())`，比 `orElseGet` 适合“备选值本身也可能没有”的场景。`orElse`/`orElseGet` 接的是普通值，`or` 接的是另一个 `Optional`，三者按“备选值会不会为空”选，别把 `or` 当成 `orElse` 的同义词。

【小技巧】`Collectors.joining` 拼字符串不用自己循环加逗号：`collect(Collectors.joining(", "))` 直接拼，`joining(",", "[", "]")` 还能一次加上前后缀，空流时返回空字符串而不是 null。手写循环拼分隔符最容易在首尾多一个逗号，收集器把这点一次性包掉。

【小技巧】并行流里想保序用 `forEachOrdered`：普通 `forEach` 在并行流里不保证输出顺序，日志、写文件这类“顺序有意义”的终端操作要改成 `forEachOrdered`。代价是它要协调顺序、会吃掉一部分并行收益；顺序不重要时用 `forEach`，顺序重要时别为了省这点开销丢了正确性。

【小技巧】`findAny` 在并行流里比 `findFirst` 快：`findFirst` 必须尊重流的 encounter order，并行时要等前面分片确认没有更早的元素；`findAny` 哪个分片先算完就返回哪个，不用等。只要“任意一个就行”，并行流优先写 `findAny`；串行流两者没区别，写 `findFirst` 语义更直白。

【冷知识】`Optional` 没有实现 `Serializable`：它从设计上就不是用来存字段、做 DTO、放缓存的容器，序列化框架遇到它要么报错要么行为各异。原文第 4 节说“实体字段、DTO 入参不要用 Optional 包”，根子就在这里——连 JDK 自己都没让它可序列化，框架更没义务替你兜底。

【冷知识】并行流的公共池默认并行度是 CPU 核数减 1：`ForkJoinPool.commonPool()` 的并行度默认取 `Runtime.getRuntime().availableProcessors() - 1`，留一核给提交任务的线程。原文第 2 节只说并行流共用全局公共池，这个减 1 解释了为什么“8 核机器上并行流最多只有 7 个工作线程在跑”——不是池子坏了，是默认值就这么定的。

【冷知识】`groupingBy` 默认返回的是 `HashMap`，分组结果没有顺序：想按插入顺序出结果，用 `groupingBy(key, LinkedHashMap::new, downstream)` 的三参数重载显式指定 Map 工厂；并行流场景还有 `groupingByConcurrent`，返回的是 `ConcurrentHashMap`。分组后按顺序展示报表时，先看你用的是哪个重载，别指望默认的 HashMap 按你插入的顺序出数。

## 5. 面试与自测

1. 写了一串 `filter`、`map` 却没有任何输出或副作用，为什么？Stream 的执行时机是怎样的？
2. `parallelStream()` 为什么不能默认用？什么情况下才考虑？
3. `orElse` 与 `orElseGet` 有什么区别？为什么不要用 `get()` 取值？

### 参考答案

1. 中间操作是惰性求值，只登记不执行；没有终端操作（`collect`、`forEach`、`reduce`、`count` 等），整条管道一行都不会跑。
2. 它共用 ForkJoinPool 公共池，数据量小或单元素处理轻时拆分调度开销反而更慢，在里面做阻塞 IO 还会拖累同 JVM 其他并行流。只在数据量大、单元素处理重、无共享状态时才考虑，且要实测对比。
3. `orElse(默认值)` 的参数总是先求值，默认值构造贵时用 `orElseGet(() -> ...)` 惰性构造；空时该报错用 `orElseThrow`。`get()` 在空时抛 `NoSuchElementException`，和 NPE 一样难定位，不要用。

## 关联知识

- [Java](开发语言/Java/Java.md)
