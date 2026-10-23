---
知识点: Java-泛型与类型擦除
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-23
关联:
  - Java
  - MyBatis-Plus
  - ArrayList与扩容
  - ConcurrentHashMap
  - Stream与Optional
标签:
  - Java
  - 集合
  - 原理
---

# Java-泛型与类型擦除

## 核心概念
- **泛型**：把"类型"参数化，类/方法在使用时才确定操作的具体类型，编译期做类型检查。
- **类型擦除**：Java 泛型只在编译期存在，编译后类型参数被擦除——运行时 `List<String>` 和 `List<Integer>` 是同一个 `Class` 对象。这是 Java 泛型与 C# 泛型最根本的区别，也是本篇所有"坑"的根源。
- 一句话定位：泛型给你编译期的类型安全；擦除决定了运行期"看不到"这些类型——凡是需要运行期知道 T 是什么的场景（`new T`、`instanceof T`、反射取 T），都得另想办法。

## 基础知识
### 泛型类、泛型接口、泛型方法
```java
import java.util.ArrayList;
import java.util.List;

public class PageResult<T> { // 类型参数 T 在类级别声明
    private List<T> records = new ArrayList<>();
    private long total;
    public List<T> getRecords() { return records; }
    public void setRecords(List<T> records) { this.records = records; }
    public long getTotal() { return total; }
    public void setTotal(long total) { this.total = total; }
}

class GenericsUtil {
    // 泛型方法：类型参数在方法级别声明；static 方法只能用自己的类型参数
    public static <T> List<T> of(T first, T second) {
        List<T> list = new ArrayList<>();
        list.add(first);
        list.add(second);
        return list;
    }
}
```

### 边界通配符与 PECS 原则
```java
import java.util.List;

// Producer Extends：只从集合里"读"——用 ? extends（读出来一定是 Number，可安全读；不能 add）
public static double sum(List<? extends Number> list) {
    double s = 0;
    for (Number n : list) s += n.doubleValue();
    return s;
}

// Consumer Super：只往集合里"写"——用 ? super（写进去一定安全；读出来只知道是 Object）
public static <T> void fill(List<? super T> dest, T value) {
    dest.add(value);
}
```

记忆口诀：**PECS = Producer Extends, Consumer Super**。JDK 自己的例子：`Collections.copy(List<? super T> dest, List<? extends T> src)`。

### 多重边界（T 须同时满足多个边界：类边界在前、接口在后，用 & 连接）
```java
public static <T extends Number & Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

## 进阶知识
### 通配符捕获（Capture）与 helper 方法
`List<?>` 的元素是"某个未知类型"，直接 `set` 编译不过。标准写法是包一层泛型 helper，让编译器"捕获"这个未知类型：

```java
import java.util.List;

public class SwapUtil {
    public static void swap(List<?> list, int i, int j) {
        swapHelper(list, i, j); // 编译器在这里把 ? 捕获为某个具体类型 T
    }
    private static <T> void swapHelper(List<T> list, int i, int j) {
        T tmp = list.get(i); list.set(i, list.get(j)); list.set(j, tmp);
    }
}
```

### 泛型与数组：天生不合
```java
List<String>[] a1 = new List<String>[10]; // 编译错误：不允许创建泛型数组
List<?>[] a2 = new List<?>[10];           // 合法：通配符数组可以
```

原因：数组在运行期检查元素类型（协变），泛型在运行期没有类型信息，两者机制冲突（见"堆污染"）。

## 原理剖析
### 擦除规则
| 声明 | 擦除后 |
|---|---|
| 无界 `<T>` | `Object` |
| 有界 `<T extends Number>` | 第一个边界 `Number` |

```java
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; } // 调用处编译器自动插入 (T) 强转，即 checkcast
}
// 擦除后近似等于把所有 T 换成 Object
```

`String s = box.get();` 实为 `(String) box.get();`——这就是"没写转型却抛 ClassCastException"的真相（见"疑难杂症"）。

### 桥接方法（Bridge Method）
擦除会破坏多态，编译器用合成的桥接方法补上（`javap -c` 可验证，带 `ACC_BRIDGE` 标记）：

```java
class Node<T> {
    private T data;
    public void setData(T data) { this.data = data; }
}
class MyNode extends Node<Integer> {
    @Override
    public void setData(Integer data) { super.setData(data); }
    // 编译器生成桥接方法 setData(Object) 转调 setData(Integer)（擦除后父方法签名变为 setData(Object)）
}
```

坑点：反射遍历 `getDeclaredMethods()` 会看到桥接方法，框架读注解时要用 `Method.isBridge()` 过滤，或用 Spring 的 `BridgeMethodResolver`。

### 擦除带来的硬性限制
```java
public class Box2<T> {
    // private T value = new T();        // 编译错误：不能实例化类型参数
    // public boolean is(T obj) {        // 编译错误：不能对类型参数用 instanceof
    //     return obj instanceof T;
    // }
    // private static T staticField;     // 编译错误：静态上下文不能引用类的类型参数
}
```

- `new T()` 不行：运行期不知道 T 是什么。替代：传 `Class<T>` 进来调 `getDeclaredConstructor().newInstance()`，或传 `Supplier<T>` 工厂。
- `instanceof T` 不行：替代 `clazz.isInstance(obj)`。
- 静态成员不行：静态成员属于类只有一份，而 T 每个实例化都不同。

## 实战应用
### 1. MyBatis-Plus 的 BaseMapper<T> 如何反推实体类
```java
import java.lang.reflect.*;

interface BaseMapper<T> { /* … */ }
class User {}
interface UserMapper extends BaseMapper<User> { /* … */ }

// 从实现类的泛型接口签名里抠出 T；注意用 getGenericInterfaces()，
// getInterfaces() 只拿到擦除后的 Class
public static Class<?> resolveEntityClass(Class<?> mapperClass) {
    for (Type t : mapperClass.getGenericInterfaces()) {
        ParameterizedType pt = (ParameterizedType) t;
        if (pt.getRawType() == BaseMapper.class) return (Class<?>) pt.getActualTypeArguments()[0];
    }
    throw new IllegalStateException("无法解析实体类: " + mapperClass);
}
```

凡是"运行期需要知道 T"的框架代码，都是读 `getGeneric*` 系列 API——这是记忆点。

### 2. Gson / Jackson 的 TypeReference：给擦除"留条后路"
```java
import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;
import java.lang.reflect.Type;
import java.util.List;

String json = "[\"a\",\"b\"]";
// 反序列化 List<String>：直接传 List.class 会丢失元素类型
Type type = new TypeToken<List<String>>() {}.getType(); // 匿名子类的父类泛型签名写进 class 文件
List<String> list = new Gson().fromJson(json, type);
```

原理和 BaseMapper 同一招：`getGenericSuperclass()` 可读。Jackson 的 `new TypeReference<List<String>>() {}` 同理。

### 3. 集合：泛型约束的日常形态
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

List<String> names = new ArrayList<>(); // 菱形操作符，类型由左侧推断
names.add("林溯");
// names.add(123);                      // 编译期直接拦下，这就是泛型的最大价值
ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();
scores.put("沈星", 95);
```

扩容、哈希、并发细节分别见 [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md)、[HashMap](开发语言/Java/集合/HashMap.md)、[ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)。

## 疑难杂症
### 堆污染（Heap Pollution）与 @SafeVarargs
```java
import java.util.Arrays;
import java.util.List;

public class HeapPollution {
    // 可变参数 T... 本质是 T[]，而泛型数组本被禁止——可变参数开了个口子
    static void dangerous(List<String>... lists) {
        Object[] arr = lists;       // 数组协变：List<String>[] 可赋值给 Object[]
        arr[0] = Arrays.asList(42); // 运行期不报错！堆里混进了 List<Integer>
        String s = lists[0].get(0); // ClassCastException：Integer 不能转 String
    }
}
```

- 解法：方法内不往可变参数数组写"可疑"东西，并加 `@SafeVarargs`（仅 static/final 方法或构造器可用）向编译器担保安全：
  ```java
  import java.util.ArrayList;
  @SafeVarargs
  public static <T> List<T> safeOf(T... elements) {
      List<T> list = new ArrayList<>();
      for (T e : elements) list.add(e);
      return list;
  }
  ```

### ClassCastException 出现在"没写转型的地方"
```java
import java.util.ArrayList;
import java.util.List;

List raw = new ArrayList<Integer>(); // 原生类型：关掉泛型检查
raw.add("oops");                     // 编译通过（只有警告）
List<Integer> ints = raw;            // unchecked 警告
Integer i = ints.get(0);             // ClassCastException 抛在这里！
```

`ints.get(0)` 处编译器插入了 `(Integer)` 强转。排查思路：看到"没转型却报转型异常"，往上找原生类型或 unchecked 警告——污染点一定在更早的地方。

## 常见误区
- **`List<String>` 和 `List<Integer>` 运行时是同一个 Class**：`new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()` 为 `true`。别想用 `instanceof List<String>` 区分，编译直接不让过。
- **数组协变 vs 泛型不变**：`String[]` 是 `Object[]` 的子类型（运行期检查，`ArrayStoreException` 兜底）；但 `List<String>` **不是** `List<Object>` 的子类型。要"只读父类型引用"用 `List<? extends Object>`。
- **"泛型能提升性能"**：泛型是编译期机制，运行期字节码与原生类型基本一样；收益是"省掉手动强转的样板代码和 bug"，不是运行更快。

## 版本差异
- **JDK 5**：泛型引入；擦除实现（为兼容 1.4 字节码）。
- **JDK 7**：菱形操作符 `<>`，`Map<String, List<String>> map = new HashMap<>();`，右侧类型由左侧推断。
- **JDK 8**：目标类型推断增强——`List<String> list = Collections.emptyList();` 不再需要写 `Collections.<String>emptyList()`。

## 关联知识
- [Java](开发语言/Java/Java.md)：语言基础总览。
- [MyBatis-Plus](开发语言/Java/MyBatis-Plus.md)：`BaseMapper<T>` 的泛型实体反推。
- [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md)：泛型在集合中的典型应用与扩容机制。
- [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)：双类型参数 `K, V` 的并发容器。
- [Stream与Optional](开发语言/Java/Stream与Optional.md)：Stream API 是泛型方法与通配符的重度使用者（如 `map(Function<? super T, ? extends R>)`）。
- [Java-反射与注解](开发语言/Java/Java-反射与注解.md)：`getGeneric*` 反射 API 是运行期读回泛型签名信息的唯一通道。
