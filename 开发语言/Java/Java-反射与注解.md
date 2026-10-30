---
知识点: Java-反射与注解
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java
  - Spring-IOC与AOP
  - MyBatis-Plus
  - Bean-Validation
  - JVM-类加载
标签:
  - Java
  - 原理
  - 实践
---

# Java-反射与注解

## 核心概念
- **反射**：程序在运行时检查、操作类、字段、方法、构造器的能力，入口是 `java.lang.Class`。
- **注解**：代码的元数据标记，本身不产生行为；行为由"读取注解的代码"赋予。两条读取路线：编译期（注解处理器 APT）与运行期（反射）。
- 一句话定位：注解是"贴标签"，反射是"读标签并动手"——Spring、MyBatis-Plus、Hibernate Validator 的核心机制都建立在这对组合上。

## 基础知识
### 获取 Class 对象的三种方式
```java
Class<String> c1 = String.class;                 // 类字面量：编译期已知时用它
Class<?> c2 = "hello".getClass();                // 实例获取：运行时反查
Class<?> c3 = Class.forName("java.lang.String"); // 字符串类名：框架按配置加载类时用它
```

`Class.forName()` 会触发类加载与初始化（执行 static 块），`String.class` 只加载不初始化（见 [[JVM-类加载]]）。

### getDeclared* 与 get* 的区别
| 方法 | 范围 |
|---|---|
| `getDeclaredField/Method/Constructor` | 本类声明的全部成员（含 private），不含继承 |
| `getField/Method` | 本类 + 父类中 public 的成员 |

构造器不存在继承，所以只有 `getDeclaredConstructor(s)` / `getConstructor(s)`，没有"继承版"之分。

### Field / Method / Constructor 基本操作
```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

class User {
    private String name;
    public User() {}
    public User(String name, int age) { this.name = name; }
    private void greet(String who) { System.out.println("Hi " + who + ", 我是" + name); }
}

public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        Class<User> clazz = User.class;
        User user = new User();

        Field nameField = clazz.getDeclaredField("name"); // private 字段用 getDeclared
        nameField.setAccessible(true); // 绕过访问检查（JDK 9+ 有模块化约束，见"常见误区"）
        nameField.set(user, "林溯");

        Method greet = clazz.getDeclaredMethod("greet", String.class); // 参数类型必须精确匹配
        greet.setAccessible(true);
        greet.invoke(user, "苏晚");

        // Class#newInstance() 自 JDK 9 起已废弃，改用构造器
        Constructor<User> ctor = clazz.getDeclaredConstructor(String.class, int.class);
        ctor.setAccessible(true);
        User u2 = ctor.newInstance("沈星", 24);
    }
}
```

`Class#newInstance()` 被废弃的原因：只能调无参构造器，且把构造器抛出的受检异常包装成 `InstantiationException` 等，掩盖真实异常；`getDeclaredConstructor().newInstance()` 抛出原始异常，行为更诚实。

## 进阶知识
### 注解的元注解
```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME) // SOURCE < CLASS < RUNTIME，框架用注解都是 RUNTIME 档
@Target(ElementType.METHOD)         // 可标注位置：类/方法/字段/参数/包/类型等
@Documented                         // 生成 Javadoc 时保留
@Inherited                          // 子类可继承父"类"上的注解（仅类继承，见"常见误区"）
public @interface LogTime {
    String value() default "";       // 注解成员本质是无参方法
    boolean async() default false;
}
```

- `@Repeatable`（JDK 8+）：允许同一位置重复标注同一注解，需配套一个"容器注解"。
- 注解成员类型受限：基本类型、String、Class、枚举、注解，及这些类型的一维数组。

### 注解处理器的两条路线
- **编译期 APT**（`javax.annotation.processing`）：`javac` 阶段扫描注解、生成代码或报错。Lombok、MapStruct 走这条路——零运行期开销。
- **运行期反射**：启动/运行时用 `getAnnotation()` 读取。Spring、MyBatis-Plus、Hibernate Validator 走这条路——灵活但有成本。

```java
import java.util.Set;
import javax.annotation.processing.*;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.*;
import javax.tools.Diagnostic;

// 编译期处理器骨架：扫描 @LogTime 并输出到编译日志
@SupportedAnnotationTypes("com.example.LogTime")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class LogTimeProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (Element e : roundEnv.getElementsAnnotatedWith(LogTime.class)) {
            processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "发现 @LogTime: " + e);
        }
        return true;
    }
}
```

## 原理剖析
- **反射的实现**：`Method.invoke()` 前 15 次走 native 调用（"膨胀阈值"），之后生成专用字节码加速——反射"越用越快"，但冷路径始终慢。
- **为什么慢**：每次 `invoke` 都有参数装箱/拆箱、安全检查、方法查找；且反射调用阻碍 JIT 内联（JIT 看不到稳定调用目标）。
- **注解的本质**：`@LogTime("x")` 编译后变成实现该注解接口的动态代理实例；`getAnnotation()` 拿到的是代理对象，`value()` 就是调代理方法——这解释了"注解成员为什么只能是那几种类型"。

## 实战应用
### 1. Spring IOC：反射实例化 + 注入（见 [Spring-IOC与AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)）
```java
// 伪代码：容器根据 BeanDefinition 创建 Bean
Class<?> clazz = Class.forName("com.example.UserService");
Object bean = clazz.getDeclaredConstructor().newInstance(); // 构造器注入则传参
for (Field field : clazz.getDeclaredFields()) {             // 字段注入
    if (field.isAnnotationPresent(Autowired.class)) {
        field.setAccessible(true);
        field.set(bean, getBean(field.getType()));          // 从容器取依赖注入
    }
}
```

### 2. AOP：代理如何读注解
`@Transactional`、`@Async` 本身不做事，是代理在调用前后读注解决定行为。注意 JDK 动态代理拿到的是接口方法，注解可能标在实现类方法上，所以框架用 `AnnotationUtils.findAnnotation` 而非裸 `getAnnotation`（它能处理桥接方法、接口继承等坑）：

```java
import java.lang.reflect.Method;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.transaction.annotation.Transactional;

// 伪代码：AOP 拦截器内部逻辑
public Object intercept(Object target, Method method, Object[] args) throws Exception {
    Transactional tx = AnnotationUtils.findAnnotation(method, Transactional.class);
    if (tx != null) {
        return invokeInTransaction(target, method, args); // 开启事务 -> 调用 -> 提交/回滚
    }
    return method.invoke(target, args);
}
```

### 3. MyBatis-Plus：@TableName / @TableField 的解析（见 [MyBatis-Plus](开发语言/Java/MyBatis/MyBatis-Plus.md)）
MP 启动时扫描实体类，一次性把注解解析成元数据缓存：

```java
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableName;

// 伪代码：实体类 -> 表名（显式指定优先，否则驼峰转下划线）
public static String resolveTableName(Class<?> entityClass) {
    TableName tn = entityClass.getAnnotation(TableName.class);
    return (tn != null && !tn.value().isEmpty()) ? tn.value() : toUnderline(entityClass.getSimpleName());
}

// exist=false 的字段直接排除，不参与 SQL
public static boolean isDbField(java.lang.reflect.Field field) {
    TableField tf = field.getAnnotation(TableField.class);
    return tf == null || tf.exist();
}
```

### 4. Bean-Validation：注解校验链路（见 [Bean-Validation](开发语言/Java/Bean-Validation.md)）
链路：`@NotBlank` → 元注解 `@Constraint(validatedBy=...)` 指定校验器类 → 运行期反射实例化校验器并执行。自定义约束注解也是这个套路：

```java
import java.util.Set;
import javax.validation.*;
import javax.validation.constraints.*;

public class UserForm {
    @NotBlank(message = "用户名不能为空")
    @Size(max = 20, message = "用户名最多 20 字符")
    private String name;
    public UserForm(String name) { this.name = name; }
}

Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
Set<ConstraintViolation<UserForm>> violations = validator.validate(new UserForm(""));
violations.forEach(v -> System.out.println(v.getPropertyPath() + ": " + v.getMessage()));
```

### 5. 自定义注解实战：方法耗时统计
```java
import java.lang.reflect.Method;

// LogTime 注解定义见"进阶知识"；运行期读取并执行
public class LogTimeRunner {
    public static void invokeWithLog(Object target, String methodName) throws Exception {
        Method method = target.getClass().getDeclaredMethod(methodName);
        LogTime logTime = method.getAnnotation(LogTime.class);
        long start = System.nanoTime();
        try {
            method.invoke(target);
        } finally {
            if (logTime != null) {
                System.out.printf("[%s] 耗时 %d ms%n", logTime.value(), (System.nanoTime() - start) / 1_000_000);
            }
        }
    }
}
```

## 疑难杂症
- **`NoSuchMethodException` 排查三板斧**：① private 方法只能用 `getDeclaredMethod`，别用 `getMethod`；② 参数类型精确匹配——`getMethod("setAge", int.class)` 与 `Integer.class` 是两个签名；③ 是否被桥接方法/泛型擦除干扰（擦除专题见 [Java-泛型与类型擦除](开发语言/Java/Java-泛型与类型擦除.md)）。
- **泛型擦除导致反射拿不到类型参数**：`class StringList extends ArrayList<String>`，`getSuperclass()` 只能拿到 `ArrayList.class`。要读泛型签名：
  ```java
  import java.lang.reflect.*;
  Type t = StringList.class.getGenericSuperclass(); // 带泛型签名的 Type
  Class<?> arg = (Class<?>) ((ParameterizedType) t).getActualTypeArguments()[0]; // String.class
  ```
  MyBatis-Plus 的 `BaseMapper<T>` 就是靠这招反推出实体类的。
- **`InaccessibleObjectException`（JDK 9+）**：反射访问模块未 `opens` 的包时抛出，加 `--add-opens java.base/java.lang=ALL-UNNAMED` 解决；能不用 `setAccessible` 破封装就不用。

## 常见误区
- **"反射只是慢一点"**：冷路径下反射调用比直接调用慢 1–2 个数量级，且阻碍 JIT 内联。框架做法是"启动时反射一次、运行期缓存结果"（Spring 的 BeanDefinition、MP 的 TableInfo），别在循环热点里反复 `getMethod().invoke()`。
- **`setAccessible(true)` 在 JDK 9+ 不是万能钥匙**：模块系统下强行访问未 `opens` 的包直接抛异常，JDK 8 时代的"暴力反射"代码升级 JDK 11/17 常在这里翻车。
- **`@Inherited` 只对类继承生效**：父类标了 `@Inherited` 注解，子类（类）能继承；接口实现、方法重写、字段隐藏都不继承——这是元注解语义，不是 bug。

## 版本差异
- **JDK 8**：`Class#newInstance()` 尚可用（已不推荐）；`@Repeatable` 引入。
- **JDK 9+**：`Class#newInstance()` 废弃，改用 `getDeclaredConstructor().newInstance()`；模块系统引入，`setAccessible` 受 `opens` 约束。
- **JDK 16+**：强封装成为默认，反射访问 JDK 内部 API 必须显式 `--add-opens`。

## 小技巧与冷知识

【小技巧】把查到的 `Method`、`Field`、`Constructor` 缓存起来复用，不要每次调用都重新 `getDeclaredMethod`。原文已讲框架是“启动时反射一次、运行期缓存结果”，自己写工具类时也可以用一个 `ConcurrentHashMap` 按“类名 + 方法名 + 参数类型”做键，第一次查到后放进去，后面直接取，这样既省了查找开销，也避免了在热点循环里反复做反射查找。

【冷知识】`Method.invoke()` 并不是每次都走同一条路：原文提到前 15 次走 native 调用，之后会生成专用的字节码来加速，所以同一个反射方法“越用越快”。这也意味着只调用一两次的冷路径最慢，压测时如果只测一次就下结论，很容易高估反射的真实成本。

【小技巧】用 `Class.forName(String name, boolean initialize, ClassLoader loader)` 这个三参数重载，可以只加载类而不触发初始化。原文讲过单参数的 `Class.forName()` 会执行 static 块，如果你只是想拿到 `Class` 对象做类型判断、读注解，却不想触发类的静态初始化副作用，就把第二个参数传 `false`，需要初始化时再由首次主动使用去触发。

【小技巧】在 JDK 9 及以上，判断能不能破封装时优先用 `trySetAccessible()` 而不是直接调 `setAccessible(true)`。前者是 `AccessibleObject` 上 JDK 9 引入的方法，成功返回 `true`、失败返回 `false` 而不抛异常，适合在工具代码里先探测再降级；后者在模块未 `opens` 的包上会直接抛 `InaccessibleObjectException`，原文的疑难杂症一节也提到了这个异常和 `--add-opens` 的解法。

【冷知识】`getAnnotation()` 拿到的注解对象并不是一个普通实例，而是实现该注解接口的动态代理。原文原理剖析一节讲过这一点，它带来的实际影响是：注解实例的 `equals`、`hashCode`、`toString` 都是按注解规范实现的，两个从不同类上读到的、成员值相同的同种注解可以 `equals` 为真，比较注解时不要用 `==`，也不要指望它有你自己写的实现类。

【小技巧】读方法上的注解时，涉及泛型方法或接口实现时不要只用裸 `getAnnotation()`，要留意桥接方法。原文在 AOP 一节提到 Spring 用 `AnnotationUtils.findAnnotation` 而不是裸 `getAnnotation`，原因就是 JDK 动态代理拿到的是接口方法、注解可能标在实现类方法上，而且泛型擦除会生成桥接方法，直接读很可能读到桥接方法而漏掉注解；自己不用 Spring 时，至少要先判断 `method.isBridge()` 并顺着桥接关系找到原始方法再读。

【小技巧】取泛型父类的实际类型参数时，用 `getGenericSuperclass()` 而不是 `getSuperclass()`。原文疑难杂症一节给过例子：`class StringList extends ArrayList<String>`，`getSuperclass()` 只能拿到 `ArrayList.class`，而 `getGenericSuperclass()` 返回的 `ParameterizedType` 上用 `getActualTypeArguments()[0]` 才能拿到 `String.class`，MyBatis-Plus 的 `BaseMapper<T>` 反推实体类就是这个套路；注意只有写在类声明里的具体类型参数才能这样拿到，运行时 `new ArrayList<String>()` 的泛型信息是擦除掉的。

【冷知识】`@Inherited` 只对“类继承类”生效，接口实现、方法重写、字段隐藏都不继承。原文常见误区一节明确讲过这一点，容易被忽略的是：子类继承到的其实是父类上的那一个注解实例，`getAnnotation()` 在子类上能读到它，但 `getDeclaredAnnotation()` 在子类上读不到，因为后者只看本类直接标注的注解，排查“注解怎么读不到”时先分清这两个方法。

【小技巧】读可重复注解时用 `getAnnotationsByType()`，不要自己去拆容器注解。JDK 8 引入 `@Repeatable` 后，同一位置重复标注的注解在编译后会被包进一个容器注解里，直接 `getAnnotation(容器.class)` 才能看到容器，`getAnnotation(原注解.class)` 拿不到；`getAnnotationsByType()` 会自动把容器拆开返回数组，写通用注解扫描工具时用它最省事，也最不容易漏。

【冷知识】注解的保留级别只有 `RUNTIME` 才能被反射读到，`CLASS` 级别的注解虽然会进 class 文件，但运行期用反射是读不到的。原文讲过保留级别是 SOURCE < CLASS < RUNTIME，框架用注解都是 RUNTIME 档；反过来，如果你只想让注解参与编译期检查或字节码分析而不想被运行期读到，用 `CLASS` 就够了，不要误以为进了 class 文件就等于反射可见。

【小技巧】操作数组不要手写 `Object[]` 强转，用 `java.lang.reflect.Array` 这个专用工具类。反射拿到的数组字段可能是 `int[]`、`String[]` 等任意类型，`Array.getLength()`、`Array.get()`、`Array.newInstance(componentType, length)` 都能统一处理基本类型数组和对象数组，避免了为每种数组类型写分支，也避免了把 `int[]` 当成 `Object[]` 强转时抛 `ClassCastException`。

【冷知识】类的 `getName()` 对数组返回的是 JVM 内部名而不是源码写法，例如 `String[].class.getName()` 是 `[Ljava.lang.String;`，而 `int[].class.getName()` 是 `[I`。日志、报错信息或按类名做映射时如果直接用 `getName()`，数组类型会显得很奇怪，需要给人看的名字用 `getCanonicalName()`，它返回的就是源码里写的 `java.lang.String[]` 这种形式。

## 面试与自测

1. 要读本类声明的 private 方法用 `getMethod` 还是 `getDeclaredMethod`？要读父类的 public 方法又用哪个？构造器有没有“继承版”之分？
2. 框架用的注解为什么都要 `@Retention(RetentionPolicy.RUNTIME)`？编译期 APT 路线和运行期反射路线各有什么取舍？
3. 反射调用为什么慢？框架一般用什么办法规避这个成本？

### 参考答案

1. 本类声明的全部成员（含 private）用 `getDeclaredField/Method/Constructor`，不含继承；本类 + 父类中 public 的成员用 `getField/Method`。构造器不存在继承，所以只有 `getDeclaredConstructor(s)` / `getConstructor(s)`，没有“继承版”之分。排查 `NoSuchMethodException` 的第一板斧就是：private 方法只能用 `getDeclaredMethod`，别用 `getMethod`。
2. 保留级别是 SOURCE < CLASS < RUNTIME，框架用注解都是 RUNTIME 档，因为运行期反射要靠 `getAnnotation()` 读到它。编译期 APT（如 Lombok、MapStruct）在 `javac` 阶段扫描注解生成代码或报错，零运行期开销；运行期反射（如 Spring、MyBatis-Plus、Hibernate Validator）启动/运行时读取，灵活但有成本。
3. 每次 `invoke` 都有参数装箱/拆箱、安全检查、方法查找，且反射调用阻碍 JIT 内联（JIT 看不到稳定调用目标）；冷路径下比直接调用慢 1–2 个数量级。框架的做法是“启动时反射一次、运行期缓存结果”（如 Spring 的 BeanDefinition、MP 的 TableInfo），别在循环热点里反复 `getMethod().invoke()`。

## 关联知识
- [Spring-IOC与AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)：IOC 容器与 AOP 代理是反射+注解最重的两个使用者。
- [MyBatis-Plus](开发语言/Java/MyBatis/MyBatis-Plus.md)：`@TableName`、`@TableField`、`BaseMapper<T>` 的注解解析与泛型反推。
- [Bean-Validation](开发语言/Java/Bean-Validation.md)：约束注解的校验链路与自定义约束。
- [JVM-类加载](开发语言/Java/JVM/JVM-类加载.md)：`Class.forName` 触发的加载、链接、初始化过程。
- [Java](开发语言/Java/Java.md)：语言基础总览。
- [Java-泛型与类型擦除](开发语言/Java/Java-泛型与类型擦除.md)：桥接方法、类型擦除对反射遍历的干扰与过滤。
