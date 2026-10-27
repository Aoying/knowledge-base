---
知识点: Java 新特性
层级: 基础
适用版本: JDK 17 / 21
更新日期: 2026-10-24
关联:
  - Java
  - 虚拟线程
  - Stream与Optional
  - JVM-字节码与JIT
标签:
  - Java
  - 原理
  - 实践
---

# Java 新特性

## 1. 核心概念

从 JDK 8 基线看 17/21，三条主线：**数据建模**（record、sealed）、**模式匹配**（instanceof、switch、record 模式）、**并发模型**（虚拟线程）；其余是可读性改进。判断标准：能不能让银行后端代码少写样板、少出状态错误。

| 特性 | 正式版本 | JEP |
|---|---|---|
| var 局部变量推断 | 10 | 286 |
| switch 表达式 | 14 | 361 |
| 文本块 | 15 | 378 |
| instanceof 模式匹配 | 16 | 394 |
| record | 16 | 395 |
| sealed 类 | 17 | 409 |
| switch 模式匹配 | 21 | 441 |
| record 模式 | 21 | 440 |
| 虚拟线程 | 21 | 444 |
| Sequenced Collections | 21 | 431 |
| String Templates | 21 仅预览 | 430，未正式，不要生产用 |

## 2. 基础知识

**var（10）**：只限局部变量，编译期推断，不是动态类型。

```java
// 旧
Map<String, List<Order>> map = new HashMap<String, List<Order>>();
// 新：类型一眼可见时才用；返回类型不明显时别用 var 藏信息
var map2 = new HashMap<String, List<Order>>();
```

**switch 表达式（14）**：有返回值、不用 break、`->` 分支不穿透。

```java
// 旧
String desc;
switch (status) { case 1: desc = "待审"; break; default: desc = "未知"; }
// 新
String desc2 = switch (status) {
    case 1 -> "待审";
    case 2 -> "通过";
    default -> "未知";
};
```

**文本块（15）**：写 SQL、JSON 不再拼加号，注意内容以换行开头的写法。

```java
String sql = """
        SELECT id, amt FROM loan_apply
        WHERE status = ?
        """;
```

**instanceof 模式匹配（16）**：判断即绑定，省掉强转。

```java
// 旧
if (obj instanceof String) { String s = (String) obj; use(s); }
// 新
if (obj instanceof String s) { use(s); }
```

## 3. 进阶知识

**record（16）**：不可变数据载体，编译器生成构造器、访问器、`equals/hashCode/toString`。组件访问器是 `name()` 不是 `getName()`。

```java
// 旧 DTO：字段+构造器+getter+equals/hashCode 约 50 行
// 新
public record LoanApplyDTO(Long id, BigDecimal amt, String status) {
    public LoanApplyDTO {  // 紧凑构造器：只做校验，不写 this.x =
        if (amt == null || amt.signum() < 0) throw new IllegalArgumentException("金额非法");
    }
}
```

**sealed 类（17）**：用 `permits` 封闭继承范围，子类必须是 `final`、`sealed` 或 `non-sealed`。

```java
public sealed interface ApproveState permits Pending, Approved, Rejected {}
public record Pending(String auditor) implements ApproveState {}
public record Approved(Instant at) implements ApproveState {}
public record Rejected(String reason) implements ApproveState {}
```

**switch 模式匹配 + record 模式（21）**：对 sealed 类型做穷尽匹配，漏一个状态编译报错；record 模式直接解构组件，可带 `when` 守卫。

```java
String msg = switch (state) {
    case Pending p            -> "待审，审批人=" + p.auditor();
    case Approved(var at)     -> "已通过 " + at;          // record 模式解构
    case Rejected r when r.reason().isBlank() -> "驳回（无原因）";
    case Rejected r           -> "驳回：" + r.reason();
    // sealed 已穷尽，不需要 default
};
```

**Sequenced Collections（21）**：`SequencedCollection/Map/Set` 统一首尾操作：`addFirst/addLast/getFirst/getLast` 等及反序视图 `reversed()`；`List`、`Deque`、`LinkedHashMap` 等都实现了它。

**虚拟线程（21）**：本篇只给结论——IO 密集请求收益大、不是线程池。创建、pinning、连接池注意点全在 [虚拟线程](开发语言/Java/虚拟线程.md)，不重复。

## 4. 原理剖析

这些特性大多是**编译期语法 + 少量标准库/字节码约定**，不是运行时魔法：

- var、文本块、switch 表达式编译后基本等价于旧写法，无运行时代价。
- record 是 `final` 类，字段 `private final`、没有 setter，`equals/hashCode` 由 `invokedynamic` 按组件生成。字节码细节见 [JVM 字节码与 JIT](开发语言/Java/JVM/JVM-字节码与JIT.md)。
- 模式匹配编译为类型检查 + 转换，switch 靠 typeSwitch 分发；穷尽性在编译期检查，所以 sealed 新增子类后旧 switch 直接编译失败——这是它建模状态机的价值。

## 5. 实战应用

银行后端「值不值得用」的判断：

| 特性 | 判断 |
|---|---|
| record | **推荐**：接口入参/出参 DTO、内部查询结果、事件对象。**不要**用在 JPA/MyBatis 实体上——实体需要可变字段和框架反射填充，和不可变 record 冲突；MyBatis 可映射到 record 的查询结果是例外，需逐个验证 |
| sealed + switch 模式匹配 | **推荐**：审批状态、单据状态这类封闭状态集建模，比枚举多带数据、比继承开放体系多穷尽检查。状态散落在字符串字段里的老代码，新模块先从这里改 |
| 文本块 | **推荐**：原生 SQL、JSON 模板、测试数据 |
| instanceof / record 模式 | **推荐**：消灭强转和重复解构，低风险渐进替换 |
| var | **克制**：只在右侧类型显而易见时用，链式调用、字面量场景别用 |
| Sequenced Collections | **按需**：LRU、队列型结构受益，一般业务集合不必为它改代码 |
| String Templates | **不用**：21 里只是预览（需 `--enable-preview`），后续版本已调整方向，别写进生产代码 |

Stream 侧的配合用法（`toList()` 不可变结果等 JDK 16 起的小改进）见 [Stream 与 Optional](开发语言/Java/Stream与Optional.md)。

## 6. 疑难杂症

- **record 当实体改不动字段**：record 不可变是设计而非 bug。要「改」状态就新建对象，或该类本就不该是 record。
- **sealed 子类编译报错 `class is not allowed to extend sealed class`**：子类没写进 `permits`，或子类没声明 `final/sealed/non-sealed` 三者之一。
- **switch 穷尽还要求 default**：匹配的不是 sealed 类型（普通接口/类编译器无法证明穷尽），或类路径上是旧版本编译的 sealed 类。
- **开启预览特性后换 JDK 跑不了**：预览特性与 class 文件版本绑定，`--enable-preview` 编出的字节码换版本直接拒绝加载——这也是 String Templates 不进生产的原因。

## 7. 常见误区

- 「var 是弱类型」：错。var 是静态推断，编译后类型固定，`var x = null` 直接编译失败。
- 「record 就是 Lombok @Data」：错。record 不可变、无 setter、访问器无 `get` 前缀；Lombok 生成的是可变 Bean，两者定位不同。
- 「升到 21 就该全面拥抱预览特性」：错。预览特性 API 可能变、二进制不兼容，生产只用正式特性。
- 「新特性 = 要重写老代码」：错。以上特性全部可增量使用，老代码不动也能编译运行。

## 8. 性能优化

语言特性本身几乎不带来性能变化，别指望换语法提速。21 的性能收益主要在两处：一是虚拟线程提升 IO 密集服务的并发承载（见 [虚拟线程](开发语言/Java/虚拟线程.md)）；二是 GC 进步——G1 自 JDK 9 起是默认 GC，21 主要是持续调优；分代 ZGC 在 21 以 JEP 439 落地（` -XX:+UseZGC -XX:+ZGenerational` 显式开启，不是默认），低延迟场景可选。GC 选型与调优见 [JVM-GC 调优](开发语言/Java/JVM/JVM-GC调优.md)，本篇不展开。

## 9. 版本差异

**两个 LTS 站点**：17 = record 收尾 + sealed 正式，模式匹配只到 instanceof；21 = 模式匹配补全（switch + record 模式）+ 虚拟线程 + Sequenced Collections。只记两站即可，中间的非 LTS 版本（18–20）全是这些特性的预览期，不必单独学。

**迁移专节（结合现项目 Spring Boot 2.7.3，必须分清两件事）**：

1. **只用 17 的语言特性**：只需把 JDK 升到 17。Spring Boot 2.7 支持在 JDK 17 上运行（其基线仍是 JDK 8），record、sealed、文本块都能用，框架不用动。风险点是老字节码库、旧 JDBC 驱动在 17 强封装下的反射报错，先在测试环境跑通。
2. **要用虚拟线程等 Web 层新能力**：必须升 Spring Boot 3.x（`spring.threads.virtual.enabled` 是 Boot 3.2+ 的能力，且要求 JDK 21）。这一步的最大坑是 **Jakarta 命名空间迁移**：Boot 3 基于 Jakarta EE 9+，`javax.servlet.*`、`javax.persistence.*`、`javax.validation.*` 等要整体改成 `jakarta.*`，漏改的表现是编译不过或启动时类找不到。注意只有 Jakarta EE 的包改名，JDK 自带的 `javax.sql`、`javax.crypto` 等**不改**，别全局替换误伤。配套的还有 Spring Framework 6、Security 配置 API 变化，工作量按「一次大版本升级」评估，不要和升 JDK 混为一谈。

语言基础总览见 [Java](开发语言/Java/Java.md)。

## 10. 面试与自测

- record 为什么不能做 JPA 实体？紧凑构造器能做什么、不能做什么？
- sealed 相比枚举和普通继承各强在哪？穷尽检查靠什么实现？
- switch 模式匹配里 `when` 守卫和嵌套 if 有何区别？分支顺序为什么重要？
- String Templates 为什么不能在生产用？预览特性的二进制代价是什么？
- 项目在 Boot 2.7，用上 sealed 需要动框架吗？用上虚拟线程呢？

### 参考答案

1. record 是不可变数据载体：final 类、字段 private final、没有 setter，编译器生成构造器、访问器与 equals/hashCode/toString，组件访问器是 `name()` 而不是 `getName()`。JPA/MyBatis 实体需要可变字段和框架反射填充，与不可变 record 冲突，所以不能做实体；紧凑构造器只能做校验，不能写 `this.x = ...` 赋值，字段赋值由编译器自动完成。
2. 相比枚举，sealed 类型每个状态可以是携带不同数据的 record，表达力更强；相比普通开放继承，sealed 用 `permits` 封闭继承范围，子类必须是 final、sealed 或 non-sealed 之一。穷尽检查靠编译期检查实现：对 sealed 类型做 switch 模式匹配时漏一个状态就编译报错，sealed 新增子类后旧 switch 也会直接编译失败，这正是它建模审批、单据等封闭状态集的价值。
3. `when` 守卫是分支匹配条件的一部分：守卫不满足时该分支不算匹配，会继续尝试下一个分支；而嵌套 if 是已经进入分支后的内部判断，条件不满足仍停留在该分支内。分支顺序因此重要：更具体、带守卫的分支要放在前面，如「驳回且原因空白」要放在普通「驳回」之前，否则通用分支会先匹配、把带守卫的分支吞掉。
4. String Templates 在 JDK 21 里只是预览特性（JEP 430，未正式），需要 `--enable-preview` 才能用，API 可能变化、二进制不兼容。预览特性与 class 文件版本绑定，用 `--enable-preview` 编出的字节码换版本直接拒绝加载，所以不能写进生产代码。
5. 用 sealed 不需要动框架：它只是 JDK 17 的语言特性，只需把 JDK 升到 17，Spring Boot 2.7 支持在 JDK 17 上运行（其基线仍是 JDK 8），record、sealed、文本块都能用，风险点是老字节码库、旧 JDBC 驱动在 17 强封装下的反射报错。用虚拟线程则必须升 Spring Boot 3.x（`spring.threads.virtual.enabled` 是 Boot 3.2+ 的能力，且要求 JDK 21），最大坑是 Jakarta 命名空间迁移：`javax.servlet.*`、`javax.persistence.*`、`javax.validation.*` 等要整体改成 `jakarta.*`，而 JDK 自带的 `javax.sql`、`javax.crypto` 等不改，别全局替换误伤；这是一次大版本升级，不能和升 JDK 混为一谈。

## 11. 关联知识

- [Java](开发语言/Java/Java.md)
- [虚拟线程](开发语言/Java/虚拟线程.md)
- [Stream 与 Optional](开发语言/Java/Stream与Optional.md)
- [JVM 字节码与 JIT](开发语言/Java/JVM/JVM-字节码与JIT.md)
- [JVM-GC 调优](开发语言/Java/JVM/JVM-GC调优.md)
