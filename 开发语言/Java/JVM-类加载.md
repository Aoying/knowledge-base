---
知识点: JVM 类加载
层级: 进阶
适用版本: JDK 8+（涉及 JDK9 模块化变化时注明）
更新日期: 2026-10-22
关联:
  - Java
  - JVM-内存模型
  - JVM-GC调优
标签:
  - Java
  - JVM
  - 实践
---

# JVM 类加载

## 1. 核心概念
- 类加载：把 `.class` 字节码读入内存，生成 `java.lang.Class` 对象。
- 类的唯一性 = 类加载器 + 全限定名共同决定，两者任一不同即为不同类。
- 生命周期：加载 → 连接 → 初始化 → 使用 → 卸载。
- 连接分三步：验证、准备、解析，三步可交叉，解析可延后到初始化后。

| 阶段 | 做什么 | 关键点 |
| --- | --- | --- |
| 加载 | 读字节码、建 Class 对象、建方法区结构 | 来源：文件、jar、网络、动态代理 |
| 验证 | 校验字节码、符号引用、栈映射 | 防止恶意字节码破坏 JVM |
| 准备 | 为静态变量分配内存并赋零值 | 仅分配 + 零值，不执行赋值 |
| 解析 | 符号引用转为直接引用 | 可延后到首次使用时才解析 |
| 初始化 | 执行 `<clinit>`：静态块 + 静态赋值 | 父类先于子类初始化 |
| 使用 | new、调用方法、读写字段 | 正常运行期 |
| 卸载 | 类加载器被回收时卸载 | 需 Class、实例、加载器均无引用 |

## 2. 基础知识
### 2.1 三层类加载器
| 加载器 | 实现 | 加载范围 | getClassLoader() |
| --- | --- | --- | --- |
| Bootstrap | C++，JVM 内部 | JDK 核心：`java.*`，JDK8 为 `rt.jar` | `null` |
| Extension / Platform | Java，JDK9 改名 Platform | JDK8：`ext` 目录；JDK9：平台模块 | 非 null |
| Application | Java | `classpath` / `-cp` | 默认上下文加载器 |

- 打印父链：
```java
ClassLoader cl = Demo.class.getClassLoader();
while (cl != null) {
    System.out.println(cl);
    cl = cl.getParent();
}
System.out.println("Bootstrap: null 表示顶层");
```
### 2.2 初始化触发：主动引用
满足任一才执行 `<clinit>`：
- `new` 创建对象。
- 读写非 `final` 静态字段，如 `Demo.count` 的 getstatic / putstatic。
- 调用静态方法，如 `Demo.run()` 的 invokestatic。
- 反射：`Class.forName("com.example.Demo")`，默认 `initialize=true`。
- 初始化子类时，先初始化父类。
- 含 `main` 方法的类，JVM 启动时初始化。
- JDK7+：`MethodHandle` 解析时涉及的类。
### 2.3 被动引用：不触发初始化
```java
class Parent { static int v = 10; static { System.out.println("Parent init"); } }
class Child extends Parent { static { System.out.println("Child init"); } }
// 1. 引用父类字段，只初始化 Parent，不初始化 Child
System.out.println(Child.v);
// 2. 数组声明，只加载不初始化
Parent[] arr = new Parent[10];
// 3. 编译期常量直接进常量池，不初始化（需 final + 编译期可定值）
class Const { static final int K = 3; static { System.out.println("init"); } }
System.out.println(Const.K); // 不打印 init
```

## 3. 进阶知识
### 3.1 静态赋值时机
| 阶段 | 动作 | 例：`static int a = 5;` |
| --- | --- | --- |
| 准备 | `a = 0` | 零值：int→0、引用→null、boolean→false |
| 初始化 | `a = 5` | 按源码顺序合并进 `<clinit>` 执行 |

- 例外：`static final` 编译期常量在准备阶段直接赋真实值。
### 3.2 经典输出题：计数器顺序
```java
public class Singleton {
    private static Singleton inst = new Singleton(); // 先于 count 声明
    public static int count1;
    public static int count2 = 0;
    private Singleton() { count1++; count2++; }
    public static Singleton get() { return inst; }
    public static void main(String[] args) {
        Singleton.get();
        System.out.println(Singleton.count1); // 1
        System.out.println(Singleton.count2); // 0
    }
}
```
- 解析：`<clinit>` 按顺序执行，先构造 `inst` 使两计数变 1。
- 随后 `count2 = 0` 显式赋值覆盖为 0，`count1` 无赋值保留 1。

## 4. 原理剖析
### 4.1 双亲委派：`loadClass` 流程
```java
// ClassLoader.loadClass 简化逻辑，勿重写它，优先重写 findClass
protected Class<?> loadClass(String name, boolean resolve) {
    Class<?> c = findLoadedClass(name);   // 1. 查缓存，已加载直接返回
    if (c == null) {
        c = (parent != null) ? parent.loadClass(name) // 2. 委派父加载器
                             : findBootstrapClass(name);
        if (c == null) c = findClass(name); // 3. 父加载不到，自己加载
    }
    return c;
}
```
| 好处 | 说明 |
| --- | --- |
| 防篡改 | `java.lang.String` 永远由 Bootstrap 加载，自定义同名类不生效 |
| 避免重复 | 同一类只被上层加载一次，类型体系唯一 |
| 安全分层 | 核心类、扩展类、应用类职责隔离 |

### 4.2 打破双亲委派
| 场景 | 打破方式 | 说明 |
| --- | --- | --- |
| SPI / JDBC | 线程上下文加载器 TCCL | 接口在 Bootstrap，实现在应用 classpath |
| Tomcat | WebAppClassLoader 优先自己 | 每 webapp 隔离 + 热部署，核心类除外 |
| OSGi / 热部署 | 网状委派、自研加载器 | 按 bundle 依赖委派，点到为止，生产慎用 |

- JDBC 例：`DriverManager` 由 Bootstrap 加载，`java.sql.Driver` 接口也在核心层。
- 驱动实现如 `com.mysql.cj.jdbc.Driver` 在应用层，父加载器看不见。
- 解法：`Thread.currentThread().getContextClassLoader()` 取应用加载器反向加载。
```java
// ServiceLoader 本质即用 TCCL 加载 META-INF/services 实现
ServiceLoader<Driver> loaders = ServiceLoader.load(Driver.class);
ClassLoader tccl = Thread.currentThread().getContextClassLoader();
```
- Tomcat：每个 webapp 一个 `WebAppClassLoader`，优先加载 `WEB-INF/classes`、`WEB-INF/lib`。
- 例外：`java.*` 等核心类仍走双亲委派，防止伪造核心类。
- 隔离收益：同名不同版本依赖可共存；卸载 webapp 即卸载其加载器实现热部署。

## 5. 实战应用
### 5.1 自定义类加载器骨架
```java
public class FileClassLoader extends ClassLoader {
    private final Path dir;
    public FileClassLoader(Path dir) { this.dir = dir; }
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            Path f = dir.resolve(name.replace('.', '/') + ".class");
            byte[] bytes = Files.readAllBytes(f);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
    public static void main(String[] args) throws Exception {
        ClassLoader loader = new FileClassLoader(Path.of("/tmp/classes"));
        Class<?> clazz = loader.loadClass("com.example.Hello");
        Object obj = clazz.getDeclaredConstructor().newInstance();
        clazz.getMethod("say").invoke(obj);
        System.out.println(clazz.getClassLoader()); // FileClassLoader 实例
    }
}
```
- 要点：只重写 `findClass`，不重写 `loadClass` 即保留双亲委派。
- 字节数组来源可换：加密文件解密、网络下载、数据库读取，逻辑不变。
### 5.2 排查命令
| 目的 | 命令 |
| --- | --- |
| 类从哪加载 JDK8 | `java -verbose:class -jar app.jar` |
| 类从哪加载 JDK9+ | `java -Xlog:class+load=info -jar app.jar` |
| 类卸载跟踪 JDK9+ | `java -Xlog:class+unload=info -jar app.jar` |
| 类层级与加载器 | `jcmd <pid> VM.class_hierarchy` |
| 系统属性与类路径 | `jcmd <pid> VM.system_properties` |
| 堆中加载器统计 | `jmap -clstats <pid>`（部分 JDK 已移除，看 jcmd 替代） |
| Arthas 看加载器 | `classloader -l`、`classloader -c <hash>`、`sc -d <类名>` |

- 代码定位来源：
```java
System.out.println(Demo.class.getProtectionDomain().getCodeSource().getLocation());
System.out.println(Demo.class.getClassLoader());
```

## 6. 疑难杂症
### 6.1 `NoClassDefFoundError` vs `ClassNotFoundException`
| 维度 | ClassNotFoundException | NoClassDefFoundError |
| --- | --- | --- |
| 类型 | 受检异常 `Exception` | `Error`，`LinkageError` 子类 |
| 时机 | 显式加载：`forName`、`loadClass` 找不到 | 编译时在、运行时缺失或 `<clinit>` 失败 |
| 常见因 | classpath 缺 jar、类名写错 | 依赖 jar 运行时缺失、静态块抛异常 |
| 排查 | 查 `-cp`、查 jar 是否含该类 | 看首个报错根因，常伴 `ExceptionInInitializerError` |

- 排查：`jar tf app.jar | grep Demo`、`unzip -l lib.jar | grep Demo` 确认类是否存在。
- 静态块失败后再访问只报 `NoClassDefFoundError`，必须翻最早一条日志找根因。
### 6.2 同名类 `ClassCastException: X cannot be cast to X`
```text
java.lang.ClassCastException: com.example.User cannot be cast to com.example.User
```
- 根因：类名相同但加载器不同，JVM 视为两个不同类型。
- 场景：热部署残留、Tomcat 与应用各放一份 jar、插件系统重复加载。
- 排查：分别打印两边 `obj.getClass().getClassLoader()` 与 `User.class.getClassLoader()` 对比。
- 解法：共享类只放父加载器可见位置，子加载器不重复放同一 jar。
### 6.3 类加载器泄漏 → Metaspace OOM
- 现象：反复热部署后 `OutOfMemoryError: Metaspace`，老类卸载不掉。
- 原因：旧加载器被引用持有 → 其全部类无法卸载 → Metaspace 只涨不降。
- 常见持有链：静态集合存对象、未清理的 `ThreadLocal`、未注销的 JDBC 驱动与监听器、未停的线程。
- 排查：`jcmd <pid> GC.class_stats`、`jmap -dump` 后用 MAT 查 `ClassLoader` 的 GC Root。
- 修复：部署卸载时清静态集合、注销驱动、停线程，`ThreadLocal` 用完 `remove()`。

## 7. 常见误区
- 误区：`Class.forName` 与 `loadClass` 相同。实际前者默认初始化，后者默认不初始化。
- 需不初始化加载时用 `Class.forName(name, false, loader)` 或 `loader.loadClass(name)`。
- 误区：准备阶段就赋源码值。实际除编译期常量外一律先零值，初始化才赋值。
- 误区：同名类能强转。实际加载器不同即不同类型，必抛 `ClassCastException`。
- 误区：重写 `loadClass` 才算自定义。实际应重写 `findClass`，重写前者易破坏委派。

## 8. 性能优化
- 减少启动期类加载：懒加载非关键类，避免启动时反射全量扫描包。
- 预热：核心链路启动后主动触发一次初始化，避免首请求承担 `<clinit>` 成本。
- 控制加载器数量：避免频繁新建加载器，防止 Metaspace 碎片与重复加载。
- 监控：`-XX:MetaspaceSize` 设下限减少早期扩容，观察 `-Xlog:class+load` 峰值。
- CDS：JDK 共享类数据归档 `java -Xshare:dump` 可加速重复启动，容器场景收益明显。

## 9. 版本差异
| 项 | JDK 8 | JDK 9+ |
| --- | --- | --- |
| 中间加载器名 | ExtClassLoader | PlatformClassLoader |
| 核心类存放 | `rt.jar`、`tools.jar` | 模块镜像 `jimage`，`rt.jar` 消失 |
| 日志参数 | `-verbose:class` | `-Xlog:class+load`（前者仍兼容） |
| 委派与模块 | classpath 全可见 | 模块封装：非导出包不可见，委派先查模块 |

- JDK9 后三层加载器仍在，但 Bootstrap 加载部分平台模块，Platform 加载其余平台模块。
- 模块化下同包拆分受限，`--add-exports` / `--add-opens` 仅作迁移期临时放行。

## 10. 面试与自测
- 类生命周期七阶段是什么？准备阶段为何只赋零值？
- 主动引用 6 种场景？被动引用 3 个例子分别说明什么？
- 双亲委派流程三步？为何说重写 `findClass` 而非 `loadClass`？
- TCCL 解决什么问题？JDBC 为何不用它不行？
- `cannot be cast to X` 同名异常的根因与定位命令是什么？
- 自测答案标准：能默写 `loadClass` 伪代码 + 手写 `findClass` 骨架即过关。

## 11. 关联知识
- JVM-内存模型：方法区 / Metaspace 是类元数据存放处，卸载与之直接相关。
- JVM-GC调优：类卸载由 GC 回收加载器触发，Metaspace OOM 需结合 GC 日志看。
- Java：反射、SPI、`ServiceLoader`、Tomcat 容器隔离均建立在类加载之上。
