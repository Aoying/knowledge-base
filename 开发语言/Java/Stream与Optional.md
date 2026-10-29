---
知识点: Stream 与 Optional
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-29
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
