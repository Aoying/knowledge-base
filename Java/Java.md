---
知识点: Java
层级: 基础
适用版本: JDK 8+ / Spring Boot 2.7
更新日期: 2026-10-22
关联:
  - Java知识点地图
  - Java-并发编程
  - 设计模式
标签:
  - Java
  - 实践
---

# Java

> 每晚 20:30 自动补充完善。贴合 Spring Boot 2.7.x + MyBatis-Plus 3.5.x 技术栈（神农 e 贷法人版同款）。

## Spring Boot 2.7 实战

### 配置加载顺序（后覆盖前）
1. 命令行参数 `--server.port=8081`
2. `SPRING_APPLICATION_JSON` / 系统环境变量
3. `application-{profile}.yml` → `application.yml`

### 常用配置片段
```yaml
server:
  port: 8080
  tomcat:
    max-threads: 200
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initial-size: 5
      max-active: 50
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 本地看 SQL
  global-config:
    db-config:
      logic-delete-value: 1
      logic-not-delete-value: 0
```

### @Transactional 失效的 4 个经典场景
1. **同类自调用**：`this` 直接调，绕过代理。解法：注入自己 / `AopContext.currentProxy()`。
2. **private / final 方法**：代理不了，不生效。
3. **异常被吞**：try-catch 吃掉异常且没抛，事务以为成功。想回滚就 `throw new RuntimeException` 或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
4. **checked 异常**：默认只回滚 RuntimeException。受检异常要回滚加 `rollbackFor = Exception.class`。

### 事务传播常用组合
- `REQUIRED`（默认）：有就加入，没有新建。
- `REQUIRES_NEW`：挂起当前，永远新建——记流水、审计日志独立提交用它。
- 注意：`REQUIRES_NEW` 内部抛异常不影响外部已提交的事务（已挂起隔离）。

## MyBatis-Plus 3.5 实战

### 插件配置（新版写法，别再用旧的 PaginationInterceptor）
```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor i = new MybatisPlusInterceptor();
        i.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        i.addInnerInterceptor(new OptimisticLockerInnerInterceptor()); // 乐观锁
        return i;
    }
}
```
注意：PageHelper 和 MP 分页插件**二选一**，混用会出两套 limit。

### 条件构造器速记
```java
// lambda 写法，防字段名写错
lambdaQuery().eq(User::getAge, 18).like(User::getName, "张").list();
// 动态条件：第一个参数 boolean
lambdaQuery().eq(StringUtils.isNotBlank(name), User::getName, name);
// update 只更新非 null 字段；想把字段置 null 用 updateWrapper.set("col", null)
```

### 常见坑
1. **逻辑删除**：`@TableLogic` 字段，查询自动带 `deleted=0`；自定义 SQL 要自己拼条件。
2. **主键策略**：`@TableId(type = IdType.ASSIGN_ID)` 雪花算法；OceanBase 下自增主键行为跟 MySQL 有差异，优先用 ASSIGN_ID。
3. **`updateById` 忽略 null**：全量更新用 `update(entity, wrapper)` 或 lambdaUpdate。
4. **大批量插入**：`saveBatch` 默认 1000 一批，调大 batchSize；超大批量考虑原生 JDBC batch。
5. **枚举映射**：`@EnumValue` 标在存库的字段上，别标错。

## 排查三板斧

1. **SQL 不对**：本地开 `log-impl: stdout`，直接看最终 SQL 和参数。
2. **N+1**：循环里调 mapper → 改成一次 `in` 查询 + 内存组装。
3. **慢接口**：先看 SQL 执行计划（`EXPLAIN`），再看是不是事务里套了远程调用（事务+Feign 长时间占用连接是大忌）。

## @Async 异步实战与坑

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Bean("bizExecutor")
    public Executor bizExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(10);
        e.setMaxPoolSize(50);
        e.setQueueCapacity(500);
        e.setThreadNamePrefix("biz-async-");
        e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        e.setWaitForTasksToCompleteOnShutdown(true); // 优雅停机：等任务跑完
        e.setAwaitTerminationSeconds(60);
        e.initialize();
        return e;
    }
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("异步任务异常 {}.{} 参数={}", method.getDeclaringClass().getSimpleName(), method.getName(), Arrays.toString(params), ex);
    }
}
```

```java
@Service
public class NotifyService {
    @Async("bizExecutor")                      // 必须指定线程池名，否则走默认
    public CompletableFuture<Void> sendSms(String mobile) {
        smsClient.send(mobile);
        return CompletableFuture.completedFuture(null);
    }
}
```

- **默认执行器是坑**：不配线程池时 Spring 用 `SimpleAsyncTaskExecutor`，每个任务起一个新线程，无上限——流量一上来线程爆炸 OOM（官方 Javadoc 明确写了不复用线程）。生产必须配 `ThreadPoolTaskExecutor`。
- **同类自调用失效**：跟 `@Transactional` 一个原理，`this.sendSms()` 绕过代理，同步执行。解法：注入自己调，或把异步方法拆到另一个 Bean。
- **void 方法异常会"消失"**：调用方拿不到异常，只能靠 `AsyncUncaughtExceptionHandler` 记日志；要感知结果就返回 `CompletableFuture`。
- **事务不传播**：`@Async` 方法里拿不到调用方的事务，新线程 = 新事务；需要事务就自己加 `@Transactional`。
- 拒绝策略别用默认 `AbortPolicy`（直接抛异常丢任务），IO 型业务一般用 `CallerRunsPolicy` 让调用线程兜底执行，天然限流。

## 参数校验 Bean Validation 实战

```xml
<!-- Boot 2.3+ 必须显式加，web starter 里没有了 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
@Data
public class LoanApplyDTO {
    @NotBlank(message = "客户名称不能为空")
    private String custName;
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式错误")
    private String mobile;
    @DecimalMin(value = "10000", message = "金额不能小于1万")
    private BigDecimal amount;
    @Valid // 嵌套对象级联校验，别漏
    private List<@NotBlank String> materials;
}

@PostMapping("/apply")
public R<String> apply(@Valid @RequestBody LoanApplyDTO dto) { ... }

// 统一异常返回
@RestControllerAdvice
public class ValidateHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public R<?> handle(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return R.fail("参数校验失败: " + msg);
    }
}
```

- `@Valid` 只在 Controller 入参/ `@Validated` 的 Service 方法上才触发；DTO 字段上的注解本身不会"自动"生效。
- 嵌套集合校验要双写：字段上 `@Valid` + 泛型上注解（`List<@NotBlank String>`），否则集合元素不校验。
- 新增/修改共用 DTO 时用分组：`@Validated(Create.class)` / `@Validated(Update.class)`，`@NotNull(groups = Update.class)` 按场景生效。
- 校验是第一道防线，不是唯一防线：金额、状态机流转这类业务规则放 Service 里再查一遍。

## Spring Boot 2.7 → 升级路线图（2026-10-20 核验）

| 线 | 状态 |
|---|---|
| 2.7.x（神农在用） | 2023-11 起无开源安全补丁，裸奔中 |
| 3.5.x | 开源支持止于 2026-06-30 |
| 4.0.x | 2025-11 发布，支持到 2026-12，别停在这 |
| 4.1.x | 2026-06 发布，支持到 2027-07，升级目标定它 |

- **现实路径**：2.7 → 3.5（JDK 17）→ 4.1，别想一步到位；先在 3.5 上把 deprecation warning 清零（4.0 删掉了所有 3.x 标记废弃的 API）。
- **三件硬骨头**：① JDK 8→17（`--release`、反射/封装 API）；② `javax.*` → `jakarta.*` 全量改包名；③ 依赖对齐——Flowable 6.7.2 只能跑 Boot 2.7，上 Boot 3 必须升 Flowable 7.x；MyBatis-Plus / Druid 也要换支持 Boot 3（Jakarta）的版本，升级前逐个核对 release notes（待验证）。
- **收益对得上神农这种 IO 密集型**：Boot 3.2+ 虚拟线程一键开（`spring.threads.virtual.enabled=true`），Tomcat 请求线程换虚拟线程，调 OceanBase/下游接口的等待不再占平台线程——和笔记里"虚拟线程"一节呼应。

> 来源（2026-10-20 核验）：[spring-boot-4](https://www.danvega.dev/blog/spring-boot-4-is-here)；[spring-boot-3.5 EOL](https://dev.to/feezan_khattak/spring-boot-35-stopped-getting-patches-in-june-dont-migrate-to-40-4ba)；[boot-4.0.8](https://spring.io/blog/2026/08/20/spring-boot-4-0-8-available-now/)；[SimpleAsyncTaskExecutor Javadoc](https://docs.spring.io/spring/docs/5.3.24/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html)；[Flowable 7.0.0](https://www.flowable.com/blog/releases/flowable-open-source-7-0-0-release)

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

## 虚拟线程（Java 21+，值得升级的理由）

```java
// 以前：平台线程 1:1 内核线程，开几千个就 OOM
// 现在：虚拟线程轻量到可以"一个请求一个线程"
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10000; i++)
        exec.submit(() -> callDownstream());
}
```

- 写同步代码，跑异步性能：`Thread.sleep`/`Lock` 在虚拟线程里不会 pin 住载体线程（`synchronized` 块里别调阻塞方法，仍会 pin）。
- 神农这种 IO 密集型（调 OceanBase/Redis/下游接口）最吃香，Tomcat 换虚拟线程执行器吞吐直接涨。
- 升级路径：JDK 17→21 主要是 `--release` 和依赖兼容性，先在测试环境跑一遍。

## Druid 监控与慢 SQL 定位

```yaml
# application.yml
spring:
  datasource:
    druid:
      filter:
        stat: { enabled: true, slow-sql-millis: 1000, log-slow-sql: true }
        wall: { enabled: true }   # 防火墙：防注入
```
```java
// 代码里拿慢 SQL TopN
DruidStatManagerFacade facade = DruidStatManagerFacade.getInstance();
List<Map<String,Object>> slow = facade.getDataSourceStatDataList(1).get(0)...;
```

- 浏览器开 `/druid/index.html` 看 SQL 监控页：执行次数、慢 SQL、 max 并发，一目了然（生产记得加账号密码）。
- 慢 SQL 定位链路：druid 慢日志 → `EXPLAIN` 看执行计划 → 索引/分页优化 → 回 druid 验证耗时下降。
- 神农这种金融项目：`slow-sql-millis` 设 500ms，定时扫监控页，比等用户投诉强。

## 多数据源 dynamic-datasource

```yaml
spring:
  datasource:
    dynamic:
      primary: master
      datasource:
        master: { url: jdbc:mysql://.../db1, username: root, password: xxx }
        oceanbase: { url: jdbc:mysql://.../db2, username: root, password: xxx }
```
```java
@DS("oceanbase")   // 方法/类上一个注解就切库
public List<Order> listFromOb() { return mapper.selectList(null); }
```

- 依赖：`com.baomidou:dynamic-datasource-spring-boot-starter`（和 MyBatis-Plus 同门）。
- 事务注意：`@DS` + `@Transactional` 混用时事务只管主库，跨库事务得上 Seata，别想当然。
- 神农 e 贷这种"MySQL + OceanBase"双跑的场景就靠它。

## Flowable 实战

见 [工程实践/Flowable.md](工程实践/Flowable.md)，Java 侧的调用、驳回、网关、会签、异步 job 全在那边沉淀。

## JVM GC 调优速查

```bash
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar
```

- **Xms=Xmx**：别让堆动态伸缩，伸缩时的 Full GC 最伤。
- **G1** 是 8+ 默认，`MaxGCPauseMillis=200` 给目标值（不是保证值）。
- 看 GC 日志：`-Xlog:gc*:file=gc.log:time`，找"停顿>1s"和"Full GC 频繁"。
- 神农这种长跑服务：先保证堆够（`jstat -gc` 看 Old 区涨速），再调暂停目标，别反过来。

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
