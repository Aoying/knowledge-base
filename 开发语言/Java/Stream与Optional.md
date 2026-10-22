---
知识点: Stream 与 Optional
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-22
关联:
  - Java
标签:
  - Java
  - 实践
---

# Stream 与 Optional

## Stream + Optional 实战

```java
// Stream：集合管道，一行顶五行 for
List<String> names = users.stream()
    .filter(u -> u.getAge() >= 18)
    .sorted(Comparator.comparing(User::getScore).reversed())
    .limit(10)
    .map(User::getName)
    .collect(Collectors.toList());

// Optional：告别 != null 金字塔
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("未知");
```

- `map` 里别再 `return null`，用 `flatMap` + `Optional` 保持链式。
- Stream 是"用一次就废"，复用同一个 stream 会抛 `IllegalStateException`。
- 性能敏感的大循环还是老 for，Stream 的可读性优先于那点性能。

