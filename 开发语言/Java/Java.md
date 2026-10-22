---
知识点: Java
层级: 基础
适用版本: JDK 8+ / Spring Boot 2.7
更新日期: 2026-10-22
关联:
  - Java知识点地图
  - Java-并发编程
  - Spring-Boot
  - MyBatis-Plus
  - 设计模式
标签:
  - Java
  - 实践
---

# Java

> 每晚 20:30 自动补充完善。贴合 Spring Boot 2.7.x + MyBatis-Plus 3.5.x 技术栈（神农 e 贷法人版同款）。

## 知识点拆分索引

本篇为 Java 语言基础与总索引，以下知识点已拆为独立文档：

- [Spring Boot](开发语言/Java/Spring-Boot.md)
- [MyBatis-Plus](开发语言/Java/MyBatis-Plus.md)
- [Async 异步](开发语言/Java/Async-异步.md)
- [Bean Validation](开发语言/Java/Bean-Validation.md)
- [Stream 与 Optional](开发语言/Java/Stream与Optional.md)
- [虚拟线程](开发语言/Java/虚拟线程.md)
- [Druid 监控与慢 SQL 定位](开发语言/Java/Druid.md)
- [多数据源](开发语言/Java/多数据源.md)
- [JVM GC 调优](开发语言/Java/JVM-GC调优.md)
- [Java 并发编程](开发语言/Java/并发/Java-并发编程.md)
- [设计模式](开发语言/Java/设计模式.md)

原 Java.md 中的「Flowable 实战」章节已归并到 [Flowable](工程实践/Flowable.md)。

## 排查三板斧

1. **SQL 不对**：本地开 `log-impl: stdout`，直接看最终 SQL 和参数。
2. **N+1**：循环里调 mapper → 改成一次 `in` 查询 + 内存组装。
3. **慢接口**：先看 SQL 执行计划（`EXPLAIN`），再看是不是事务里套了远程调用（事务+Feign 长时间占用连接是大忌）。

## 待补充




## 疑难杂症

### 1. OceanBase 下大事务超时回滚，MySQL 上好好的
- **现象**：批量更新几万行，MySQL 正常，OceanBase 报事务超时回滚。
- **原因**：OceanBase 对大事务限制更严（undo 保留、超时阈值），默认超时参数和 MySQL 不同。
- **解决**：大批量拆小批次提交；调大 `ob_trx_timeout`；能异步就别放事务里。

### 2. 内层 try-catch 吃掉异常，外层事务不回滚
- **现象**：`@Transactional` 方法里 try-catch 了内层异常，业务失败了但数据已提交。
- **原因**：Spring 靠异常触发回滚；异常被 catch 后事务拦截器认为"正常返回"直接提交。
- **解决**：catch 后 `throw new RuntimeException(e)` 重新抛出，或手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。

### 3. Druid 连接"假死"：能连上但 SQL 一直 hang
- **现象**：服务跑着跑着部分请求卡死，重启就好。
- **原因**：数据库端杀掉了空闲连接（如 wait_timeout），但连接池不知道，拿到的是死连接。
- **解决**：Druid 开 `testWhileIdle=true` + `validationQuery=SELECT 1`，`timeBetweenEvictionRunsMillis` 小于数据库的 wait_timeout。

## 基础知识补强

### 核心概念与术语

- **JDK / JRE / JVM 三件套**：JDK 是开发工具包（含编译器）；JRE 是运行环境；JVM 是真正执行字节码的虚拟机。
- **源码到运行**：`.java` 源文件经 `javac` 编译成 `.class` 字节码，JVM 加载、校验、解释或即时编译（JIT）执行。
- **强类型静态语言**：变量类型在编译期确定，声明时必须写明类型，类型不合直接编译失败。
- **基本类型 8 个**：`byte short int long float double char boolean`，存的是值本身，不是指向对象的引用。
- **引用类型**：类、接口、数组、枚举都是引用类型，变量里存的是对象在堆上的地址。
- **值传递**：Java 只有值传递。传基本类型是复制值；传对象是复制引用值，方法内改属性会影响原对象，重新赋值不会。
- **包（package）**：类的命名空间，用目录层级隔离同名类；全限定名 = 包名 + 类名。
- **类与对象**：类是模板（属性 + 方法），对象是按模板 `new` 出来的实例，每个对象有自己的成员变量值。
- **成员变量 vs 局部变量**：成员变量有默认值（数字 0、布尔 false、引用 null）；局部变量没有默认值，用前必须赋值。
- **重载与重写**：重载是同类内同名方法参数不同；重写是子类改写父类方法，签名相同、返回值兼容。
- **多态**：父类引用指向子类对象，运行时按实际对象类型决定调哪个重写方法。
- **封装**：字段尽量 `private`，对外只暴露方法，目的是控制修改入口、方便改内部实现。
- **抽象类与接口**：抽象类是"半成品类"（可带普通方法与字段）；接口是"能力契约"（一组方法签名），一个类只能继承一个类、可实现多个接口。

### 基础语法与用法

```java
// 一个最小可运行程序：类名必须与文件名一致（public 类）
public class Hello {
    public static void main(String[] args) {   // 程序入口，签名固定
        int age = 18;                          // 基本类型变量
        long total = 100L;                     // long 字面量带 L
        double price = 9.9;
        boolean ok = age >= 18;
        String name = "ao";                    // String 是引用类型
        System.out.println(name + " age=" + age);
    }
}
```

```java
// 控制流：if / switch / for / while 四件套
if (score >= 90) {
    System.out.println("A");
} else if (score >= 60) {
    System.out.println("及格");
} else {
    System.out.println("不及格");
}

for (int i = 0; i < 3; i++) {          // 已知次数用 for
    System.out.println(i);             // 0 1 2，下标从 0 开始
}
while (n > 0) { n--; }                 // 条件循环
for (String s : names) {               // 增强 for：遍历数组/集合
    System.out.println(s);
}
```

```java
// 类、构造器、方法的最简形态
public class User {
    private String name;               // 私有字段
    private int age;

    public User(String name, int age) {  // 构造器：无返回值，名同类名
        this.name = name;                // this 指当前对象
        this.age = age;
    }

    public String getName() { return name; }        // 读
    public void setAge(int age) {                    // 写，可加校验
        if (age < 0) throw new IllegalArgumentException("age 不能为负");
        this.age = age;
    }
}
User u = new User("ao", 5);   // new = 在堆上建对象并调构造器
```

```java
// 数组与字符串基础
int[] arr = {1, 2, 3};
System.out.println(arr.length);        // 数组长度是属性 length，不是方法
String s = "Java";
System.out.println(s.length());        // 字符串长度是方法 length()
System.out.println(s.charAt(0));       // 'J'，下标从 0 开始
System.out.println("a" + 1 + 2);       // "a12"：从左到右，遇 String 后都变拼接
System.out.println(1 + 2 + "a");       // "3a"：先做数值加法
```

```java
// 异常处理骨架：能恢复就 catch，不能就往上抛
try {
    int x = Integer.parseInt("abc");   // 会抛 NumberFormatException
} catch (NumberFormatException e) {
    System.out.println("格式不对: " + e.getMessage());
} finally {
    System.out.println("无论成败都会执行，常用于释放资源");
}
```

```java
// 接口：定义能力，实现类补细节
interface Sender {
    void send(String msg);
}
class SmsSender implements Sender {
    public void send(String msg) { System.out.println("短信: " + msg); }
}
Sender sender = new SmsSender();   // 面向接口编程：引用写接口，实现可替换
sender.send("hi");
```

### 关键机制速览

- **内存分区**：堆放对象，栈放方法调用帧与局部变量，方法区放类信息与常量。
- **垃圾回收（GC）**：JVM 自动回收没有引用指向的对象，业务代码不用手动释放内存。
- **String 不可变**：字符串对象内容不可改，拼接会产生新对象；频繁拼接用 `StringBuilder`。
- **equals 与 ==**：`==` 比基本值或引用地址；`equals` 比内容是否相等，自定义类要按需重写。
- **hashCode 约定**：两个对象 `equals` 相等，则 `hashCode` 必须相等，否则放进 HashMap/HashSet 会找不到。
- **自动装箱拆箱**：`Integer x = 5;` 是编译器帮你调 `valueOf`，但装箱会产生对象，循环里大量装箱有开销。
- **泛型**：`List<String>` 在编译期做类型检查，避免强制转换；运行时类型参数被擦除，所以不能 `new T()`。
- **访问修饰符**：`public` 全开，`protected` 本包 + 子类，缺省（不写）仅本包，`private` 仅本类。
- **static**：属于类不属于对象，所有实例共享一份；静态方法里不能直接用 `this` 和实例字段。
- **final**：修饰变量不可再赋值，修饰方法不可重写，修饰类不可继承。
- **类加载顺序**：静态代码块与静态字段按书写顺序在类首次使用时执行一次；构造器在每次 new 时执行。

### 新手易错点

- **整数除法截断**：`5 / 2` 结果是 2 不是 2.5，想得小数至少一边写成 `5.0`。
- **浮点比较**：`double` 有精度误差，金额一律用 `BigDecimal`，且用字符串构造 `new BigDecimal("0.1")`。
- **用 == 比较字符串**：字面量可能因常量池"碰巧"相等，换成变量就翻车，永远用 `equals`。
- **null 调用方法**：任何引用都可能为 null，`obj.toString()` 前先判空；常量放前面写 `"x".equals(s)` 更稳。
- **数组下标越界**：合法范围是 `0` 到 `length - 1`，循环条件写 `i <= arr.length` 必炸。
- **switch 穿透**：忘记 `break` 会继续执行下一个 case，老写法尤其常见。
- **成员变量与局部变量同名**：方法内不写 `this.` 时用的是局部变量，赋值等于没赋到字段上。
- **构造器里调可重写方法**：对象还没初始化完，子类重写方法可能读到未赋值的字段。
- **整数溢出不报错**：`int` 超过约 21 亿会绕回负数，金额、计数可能为负时用 `long`。
- **import 同名类冲突**：两个包都有同名类时，只能 import 一个，另一个写全限定名。
