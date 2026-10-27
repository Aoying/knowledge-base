---
知识点: Lambda与函数式编程
层级: 基础
适用版本: 通用（Java 8+）
更新日期: 2026-10-27
关联:
  - Stream与Optional
  - Java-泛型与类型擦除
  - Java-新特性
标签:
  - Java
  - 原理
  - 实践
---

# Lambda与函数式编程

## 1. 核心概念

Lambda 表达式是 Java 8 引入的匿名函数：把「一段行为」当作值传来传去，不用再为一次性的行为写一个实现类。它的落点是**函数式接口**——只含一个抽象方法的接口（如 `Runnable`、`Comparator`、`Function`），Lambda 就是这个唯一抽象方法的实现。

```java
// 以前：为一次排序写匿名类
users.sort(new Comparator<User>() {
    @Override
    public int compare(User a, User b) {
        return Integer.compare(a.getAge(), b.getAge());
    }
});

// 现在：Lambda 一行
users.sort((a, b) -> Integer.compare(a.getAge(), b.getAge()));
```

理解 Lambda 的三个关键点：它是函数式接口的实例（编译器按上下文推断目标类型）、它不是语法糖包装的匿名类（底层用 `invokedynamic` 生成，见第 3 节）、它让「行为参数化」成为可能（把判断逻辑、转换逻辑、比较逻辑当参数传）。

## 2. 基础知识

### Lambda 语法

```
(参数列表) -> { 方法体 }
```

省略规则（按从繁到简）：

```java
// 完整写法
Function<Integer, Integer> f1 = (Integer x) -> { return x * 2; };
// 省略参数类型（编译器从 Function 的泛型推断）
Function<Integer, Integer> f2 = (x) -> { return x * 2; };
// 单条语句可省大括号与 return
Function<Integer, Integer> f3 = (x) -> x * 2;
// 单个参数可省小括号
Function<Integer, Integer> f4 = x -> x * 2;
// 无参数必须保留空括号
Runnable r = () -> System.out.println("hi");
```

### 方法引用：Lambda 的简写

当 Lambda 体只是调用一个已有方法时，用 `::` 更简洁：

| 形式 | 例子 | 等价 Lambda |
|---|---|---|
| 类名::静态方法 | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| 对象::实例方法 | `user::getName` | `() -> user.getName()` |
| 类名::实例方法 | `String::length` | `s -> s.length()` |
| 类名::new | `User::new` | `() -> new User()` |

第三种最容易看错：`String::length` 的 Lambda 是 `s -> s.length()`，第一个参数成了方法的接收者；用在 `Comparator.comparing(User::getAge)` 里同理，`User::getAge` 等价于 `u -> u.getAge()`。

### 常用函数式接口（java.util.function）

| 接口 | 方法签名 | 典型用途 |
|---|---|---|
| `Predicate<T>` | `boolean test(T t)` | filter 的判断条件 |
| `Function<T, R>` | `R apply(T t)` | map 的转换 |
| `Consumer<T>` | `void accept(T t)` | forEach 的消费动作 |
| `Supplier<T>` | `T get()` | 惰性取值、orElseGet 的默认值 |
| `UnaryOperator<T>` | `T apply(T t)` | 同类型转换（如 replaceAll） |
| `BinaryOperator<T>` | `T apply(T a, T b)` | reduce 的合并 |
| `BiFunction<T, U, R>` | `R apply(T t, U u)` | 双参数转换 |
| `Comparator<T>` | `int compare(T a, T b)` | 排序（不在 java.util.function，但在 Lambda 语境最常用） |

组合方法也值得记：`Predicate.and/or/negate` 拼条件、`Function.andThen/compose` 串转换、`Consumer.andThen` 串消费。

### 与 Stream / Optional 配合

```java
// Stream：Lambda 是中间操作与终端操作的参数
List<String> names = users.stream()
    .filter(u -> u.getAge() >= 18)
    .sorted(Comparator.comparing(User::getScore).reversed())
    .map(User::getName)
    .collect(Collectors.toList());

// Optional：Lambda 是链上的处理步骤
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElseGet(() -> "未知");
```

Stream 与 Optional 的完整用法见 [Stream与Optional](开发语言/Java/Stream与Optional.md)。

## 3. 进阶知识

### 变量捕获：为什么要求事实最终变量

```java
int factor = 2;
Function<Integer, Integer> f = x -> x * factor;
// factor = 3; // 编译报错：Lambda 捕获的局部变量必须是事实最终变量
```

原因：Lambda 的生命周期可能长于方法栈帧（如作为返回值、存进集合），方法返回后栈上的局部变量就没了。Java 的做法是让 Lambda 捕获变量的**副本**（值拷贝），而非引用——副本与原变量一旦能改动就会出现「改了原变量、副本没变」的不一致，所以编译器强制要求被捕获的局部变量事实最终（effectively final：声明后不再改赋值，不用显式写 final）。

两点推论：实例字段、静态字段可以改（Lambda 通过 `this` / 类引用访问它们，捕获的是对象引用而非字段副本）；想在 Lambda 里累计值时，用数组、`AtomicInteger` 这类「引用不变、内容可变」的容器。

### Lambda 的 this 指向

匿名内部类的 `this` 是匿名类实例，Lambda 的 `this` 是**外围对象**。在 Lambda 里写 `this` 拿到的是包着它的类实例，这点在回调里调外围方法时很顺手，但从匿名类迁移过来时别按老直觉写。

### 底层：invokedynamic，不是匿名类

Lambda 编译后不是生成一个 `Xxx$1.class`，而是在调用点留一条 `invokedynamic` 指令，运行时由 `LambdaMetafactory` 按需生成实现类。结果：首次调用有少量生成开销，之后与直接调用接近；也因为没有为每个 Lambda 生成类文件，jar 里不会多出一堆匿名类。泛型擦除对 Lambda 的影响（桥接方法、类型推断边界）见 [Java-泛型与类型擦除](开发语言/Java/Java-泛型与类型擦除.md)。

### 自定义函数式接口

```java
@FunctionalInterface
public interface CheckedFunction<T, R> {
    R apply(T t) throws Exception;  // 允许受检异常的函数式接口
}

// 把受检异常包成运行时异常，供 Stream 链使用
public static <T, R> Function<T, R> unchecked(CheckedFunction<T, R> fn) {
    return t -> {
        try {
            return fn.apply(t);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    };
}
```

`@FunctionalInterface` 注解不是必需的，但加上后编译器会校验「恰好一个抽象方法」，多写一个就编译失败——自定义接口时建议加上。

## 4. 实战应用：策略表替代 if-else

```java
// 计费策略：类型 -> 计费逻辑，一张表代替一串 if-else
Map<String, Function<Order, BigDecimal>> calc = new HashMap<>();
calc.put("NORMAL", o -> o.getAmount());
calc.put("VIP", o -> o.getAmount().multiply(new BigDecimal("0.9")));

BigDecimal fee = calc.getOrDefault(type, o -> o.getAmount()).apply(order);
```

适用边界：策略数量少、逻辑简单时 if-else 更直观；策略多、需动态注册（如从配置、数据库加载策略）时策略表胜出。策略本身变复杂（多步骤、有状态）时回到策略模式，见设计模式相关笔记。

## 5. 常见误区

- **在 Lambda 里改局部变量**：编译直接报错，见第 3 节捕获规则；要累计用 `AtomicInteger` 或数组容器。
- **受检异常直接往 Lambda 里写**：`stream().map(x -> readFile(x))` 里 `readFile` 抛受检异常时编译不通过——函数式接口的方法签名没声明该异常。用第 3 节的 `unchecked` 包装，或改用普通 for 循环。
- **Lambda 体写成 50 行方法**：Lambda 适合短行为，体一长就抽成具名方法再用方法引用；可读性优先于「一行流」。
- **以为 Lambda 每次调用都新建对象**：无捕获的 Lambda（如 `x -> x * 2`）运行时可复用单例；有捕获的每次生成新实例——高频循环里注意，但别为此提前优化。
- **方法引用与 Lambda 混淆接收者**：`User::getAge` 在 `Comparator.comparing` 里是对流中每个元素调用，不是对某个固定 user 调用，见第 2 节第三种形式。

## 6. 版本差异

- Lambda 与函数式接口是 Java 8 引入，Java 8 项目可全量使用本文内容。
- Lambda 参数上写 `var`（如 `(var x, var y) -> x + y`）是 Java 11+ 才支持，且要求所有参数统一用 var 或全不用；Java 8 项目别写。

## 7. 面试与自测

1. Lambda 捕获的局部变量为什么必须是事实最终变量？实例字段为什么不受这个限制？
2. `String::length` 作为方法引用时等价于什么 Lambda？为什么第一个参数成了方法的接收者？
3. `stream().map(x -> readFile(x))` 中 `readFile` 抛受检异常时编译为什么不通过？给出一种不改业务逻辑的解法。

### 参考答案

1. Lambda 的生命周期可能长于方法栈帧，Java 让 Lambda 捕获局部变量的副本而非引用；副本与原变量一旦能改动就会不一致，所以编译器强制事实最终。实例字段不受限是因为 Lambda 通过 `this` 捕获对象引用，访问的是对象上的字段而非栈上副本。
2. 等价于 `s -> s.length()`。`类名::实例方法` 形式下，第一个参数被当作方法的接收者（即 `this`），其余参数依次对应方法的形参。
3. 因为 `Function.apply` 的签名没有声明受检异常，Lambda 体抛出的受检异常无处声明，编译不通过。解法：自定义允许受检异常的函数式接口（如 `CheckedFunction`），再包一层转成 `RuntimeException` 的适配器供 Stream 链使用，见正文第 3 节 `unchecked` 示例。

## 关联知识

- [Stream与Optional](开发语言/Java/Stream与Optional.md)
- [Java-泛型与类型擦除](开发语言/Java/Java-泛型与类型擦除.md)
- [Java-新特性](开发语言/Java/Java-新特性.md)
