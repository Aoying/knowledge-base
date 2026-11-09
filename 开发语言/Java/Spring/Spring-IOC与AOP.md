---
知识点: Spring IOC 与 AOP
层级: 进阶
适用版本: Spring Boot 2.7.3 / Spring Framework 5.3
更新日期: 2026-11-09
关联:
  - Spring-Boot
  - Spring-事务
  - 设计模式
标签:
  - Java
  - 设计模式
  - 原理
  - 实践
---

# Spring IOC 与 AOP

## 1. 核心概念

- **IOC（控制反转）**：对象的创建与依赖组装交给容器，业务类只声明需要什么。实现方式是 DI（依赖注入）：构造器、Setter、字段三种注入点，团队规范优先构造器注入（最终字段、可测、不依赖容器也能 new）。
- **AOP（面向切面）**：把事务、日志、权限、耗时统计这类横切逻辑从业务方法里抽出来，由代理在调用前后统一织入。Spring AOP 是运行时代理实现，不是字节码织入（那是 AspectJ）。

两者关系：AOP 的代理对象本身就是 IOC 容器管理的 Bean；事务、缓存、异步等声明式能力全都建在 AOP 代理之上。

## 2. 基础知识

```java
@Service
public class ApplyService {
    private final ApplyMapper applyMapper;   // 构造器注入，不用 @Autowired 字段注入
    public ApplyService(ApplyMapper applyMapper) { this.applyMapper = applyMapper; }
}
```

切面最小示例：统一记录接口耗时。

```java
@Aspect
@Component
@Order(10) // 多个切面时的执行顺序，值小先进入、后退出
public class CostLogAspect {

    @Around("execution(* com.example..*Controller.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            log.info("{} 耗时 {}ms", pjp.getSignature().toShortString(),
                    System.currentTimeMillis() - start);
        }
    }
}
```

切入点表达式读法：`execution(返回类型 包..类.方法(参数))`，`..` 表示任意子包，`*` 任意。通知类型：`@Before`、`@AfterReturning`、`@AfterThrowing`、`@After`、`@Around`（最强，可控制是否执行、改参、改返回值）。

## 3. 进阶知识

### 代理机制：JDK 动态代理 vs CGLIB

| 维度 | JDK 动态代理 | CGLIB |
|---|---|---|
| 原理 | 生成实现目标接口的代理类 | 生成目标类的子类 |
| 前提 | 目标必须有接口 | 类和方法不能 final |
| Spring 选择 | 有接口时默认（Boot 2.x 起默认改 CGLIB） | Boot 2.7 默认 `proxyTargetClass=true` 即 CGLIB |

关键结论：注入时用接口类型接收 Bean 时要注意实际代理类型；把代理强转成实现类会抛错；自调用绕过代理的问题两种代理都存在（见 [Spring 事务](开发语言/Java/Spring/Spring-事务.md)）。

### 切面落点（项目里真正值得抽的横切逻辑）

- **操作日志**：切 Controller/Service 公共方法，记录操作人、入参摘要、结果，落审计表（落表方法另开事务，见事务篇 REQUIRES_NEW）。
- **接口耗时与慢调用告警**：`@Around` 计时，超阈值打 WARN，与 [监控告警](工程实践/运维/监控告警.md) 的 Timer 埋点二选一，不要两头都做。
- **权限/数据权限校验**：切面里从上下文取当前用户，校验注解上声明的权限码；复杂数据权限（如只能看本机构）更适合在 SQL 层拦截，AOP 只做粗粒度功能权限。
- **幂等防重**：注解 + 切面查幂等键（Redis/唯一索引），审批提交、额度扣减这类接口必备。
- 不适合 AOP 的：业务分支判断、需要精确异常语义的逻辑——切面里吞异常会让调用方和事务都误判。

## 4. 原理剖析

### Bean 生命周期

1. **实例化**：容器按 BeanDefinition 反射调用构造器创建对象。
2. **依赖注入**：填充属性（构造器注入在此步随构造完成）。
3. **Aware 回调**：`BeanNameAware`、`ApplicationContextAware` 等把容器信息回给 Bean。
4. **BeanPostProcessor 前置**：`postProcessBeforeInitialization`，`@PostConstruct` 也在此阶段被处理。
5. **初始化**：`InitializingBean.afterPropertiesSet` / 自定义 init-method。
6. **BeanPostProcessor 后置**：`postProcessAfterInitialization`——**AOP 代理就是在这里生成的**（`AbstractAutoProxyCreator` 是 BeanPostProcessor，拿切面匹配后返回代理对象替换原 Bean）。
7. **使用与销毁**：容器关闭时走 `@PreDestroy` / `DisposableBean` / destroy-method。

时序记忆法：代理在初始化之后才生成，所以初始化回调里调用自己的事务/切面方法，织入逻辑不会生效。

### 循环依赖

A 依赖 B、B 依赖 A 时，Spring 用三级缓存解决**单例 + 字段/Setter 注入**的循环：

- 一级 `singletonObjects`：成品 Bean。
- 二级 `earlySingletonObjects`：半成品（已实例化未完成注入）的早期引用。
- 三级 `singletonFactories`：半成品的工厂，关键作用是**在需要早期暴露时提前生成代理**，保证注入的是代理而不是原始对象。

三条边界：

- **构造器注入的循环依赖无解**，启动直接报错——这是好事，说明设计上有环，拆开（抽第三个 Bean、改事件解耦）而不是换成字段注入绕过。
- `prototype` 作用域不进缓存，循环依赖同样报错。
- Boot 2.6 起默认禁止循环引用（`spring.main.allow-circular-references=false`），老项目升级后启动失败多半是撞上存量环依赖，先定位再决定拆还是临时放开。

## 5. 实战应用

- 注入方式统一构造器注入：字段加 `final`，用 Lombok `@RequiredArgsConstructor` 简化；单构造器可省 `@Autowired`。
- `@Configuration` 类默认 CGLIB 代理（`proxyBeanMethods=true`），保证 `@Bean` 方法互调拿到同一单例；只做配置组装、不互调时可关掉省启动时间。
- 切面顺序用 `@Order` 显式声明，别依赖默认顺序：常见排法是日志/链路在最外，事务次之，业务校验在内层。
- 切入点收敛到注解（如自定义 `@OpLog`）比大包表达式更可控：表达式切全包，误伤定时任务、内部组件的概率高。

## 6. 疑难杂症

- **切面不生效**：依次查——切面类是不是 Bean（漏 `@Component`）、切入点是否命中（包名/方法签名写错）、调用是不是自调用、对象是不是容器外的 `new` 出来的。
- **启动报循环依赖**：看报错链画出环，优先把环上两个类共用的逻辑抽到第三个 Service；别用 `@Lazy` 和字段注入把环藏起来，藏起来的环会在运行时以半成品状态咬人。
- **同一个请求日志打了两遍**：切面同时切了 Controller 和 Service 且表达式互相包含，或切面被注册了两次（手动 `@Bean` + 组件扫描重复）。

## 7. 常见误区

- 认为 AOP 能拦截一切：private 方法、final 方法、自调用、容器外对象都拦不到。
- 在切面里干重活（查库拼大对象写日志表）：切面在每次调用的必经之路上，开销乘以调用量。
- 混淆 AOP 与拦截器/过滤器：Filter 在 Servlet 层、Interceptor 在 MVC 层、AOP 在 Bean 方法层，粒度与可用信息不同，别互相替代。

## 8. 代理与启动成本定量

本节数字均为数量级估计，不是承诺值：基准多为 JDK 8 / Boot 1.x–2.x 时代旧机器测量，硬件、JDK 版本、切面数量都会改写绝对值，只取量级与比例。

### 8.1 代理生成：JDK 动态代理与 CGLIB

- 生成成本量级：一次代理创建（类已生成、只建实例的情形），JDK 动态代理
  约 5 μs，CGLIB 约 27.5 μs，相差约 5 倍。CGLIB 贵在字节码生成与校验，首次为某类生成子类还要付类加载与验证的一次性成本，之后同类复用缓存。
- 内存占用量级：两种代理实例本身都不大，与目标对象同量级（几十字节级的引用与回调字段）。差别在类元数据：JDK 代理按接口集合缓存一个代理类，
  同接口的多个 Bean 共用；CGLIB 每个被代理类生成一个子类，外加一个
  FastClass 分发类，即每个被代理类型至少多 2 个生成类，常驻方法区/元空间。
- 选择结论：只看生成成本，JDK 代理更便宜；但 Boot 2.7 默认 CGLIB
  （`proxyTargetClass=true`），因为它不要求目标有接口。不要为省几十微秒
  而改代理类型，接口有无与 final 约束才是决定因素。

算例 1——1000 个被代理 Bean 的生成总账：按 27.5 μs/个算，CGLIB 共
1000 × 27.5 μs = 27.5 ms；按 5.3 μs/个算，JDK 代理共 5.3 ms。差值约
22 ms，只占一次 2–3 秒启动的约 1%，代理生成从来不是启动慢的主因。

### 8.2 Bean 数量与启动时间

- 固定成本与边际成本要分开：JVM 启动、类扫描、自动配置解析是固定成本，
  约数百毫秒；每多一个 Bean 是边际成本，实例化、注入、后置处理都要付。
- Dave Syer 2018 年的 Spring Boot 启动基准（JMH，旧硬件）给出可复算的
  斜率：121 个 Bean 时中位启动 0.935 s，222 个 Bean 时 1.140 s，30 个 Bean 时 0.715 s。两两相减得边际成本约 2.0–2.4 ms/Bean，
  三点最小二乘拟合得斜率约 2.21 ms/Bean、截距（固定成本）约 655 ms。
  注意三点是不同示例应用（demo/actr/lite），类数量也随 Bean 数变化（5012→5643→6182），
  斜率里混进了类扫描的影响，只能当粗估，不能当成“每加一个 Bean 必然多 2.21 ms”。
- 用拟合式 `启动 ≈ 655 ms + 2.21 ms × Bean 数` 估算：100 个 Bean 约
  876 ms，500 个约 1.76 s，1000 个约 2.87 s，2000 个约 5.08 s。
  这是 2018 年旧基准的外推，现代 JDK 与 Boot 3.x 的绝对值会更小，
  但“固定成本 + 每 Bean 约毫秒级”的结构仍然成立。
- 扫描成本常被低估：组件扫描按类收费，不按 Bean 收费，扫描几千个类
  本身就要几百毫秒；包路径收得越宽，固定成本越高，与 Bean 数无关。

算例 2——从 500 个 Bean 涨到 1500 个：增量 1000 × 2.21 ms ≈ 2.21 s，
总启动从约 1.76 s 涨到约 3.97 s（655 ms + 2.21 ms × 1500）。
若测量涨幅远大于此，先查扫描范围与某个 Bean 的初始化耗时，
而不是数 Bean 的个数。

- 懒加载的收益有上限：Spring 官方示例中，小应用（Actuator + Security）
  开全局懒加载后启动从 2500 ms 降到 2000 ms，约省 20%。省掉的是
  Bean 创建与初始化时间，JVM 启动、扫描、Web 容器启动省不掉；
  代价是首次请求变慢、配置错误从启动期推迟到运行期。

### 8.3 AOP 方法拦截：单次调用开销

- 要分三层看开销：裸代理分发、Spring AOP 裸拦截器链、带切面逻辑的完整路径。
  裸代理分发很便宜：JDK 8 下 5000 万次调用实测，直接调用约 3.9 ns/次、
  JDK 代理约 10.8 ns/次（额外开销约 6.9 ns）、CGLIB 约 19.3 ns/次（额外开销约 15.4 ns）；
  另一份百万次调用基准给出 CGLIB 约 15 ns/次、JDK 代理约 18 ns/次，
  两份来源对谁更快结论相反，但量级一致，都是十纳秒级。
- Spring AOP 裸拦截器链要贵一个量级：走 `MethodInterceptor` 链、切点匹配
  与 `proceed()` 递归，公开基准给出约 275–500 ns/次。
  带上切面自己的逻辑后更贵：以 `@Cacheable` 缓存命中路径（含 ConcurrentMap
  查找）为例，实测约 1.5 μs/次（CGLIB 1576 ns、JDK 代理 1551 ns、AspectJ 1515 ns，
  三种实现基本相同）。也就是说链本身几百纳秒，切面逻辑常比链贵得多。
- 换算成业务体感：方法本身耗时 5 ms 时，500 ns 拦截开销占比约 0.01%，
  可以忽略；方法本身是空方法或纯内存计算（微秒级以下）且每秒调用
  百万次时，拦截开销才进占比表。

算例 3——拦截开销的累计账：按裸链 275–500 ns/次算，100 万次调用累计
275–500 ms；1 亿次调用累计 27.5–50.0 s。同一量级下，裸代理分发
100 万次只累计约 15–18 ms。带完整切面逻辑时按约 1.5 μs/次算，
100 万次累计约 1.5 s。结论：热路径上先优化切面逻辑与切点范围，
而不是纠结 JDK 代理与 CGLIB 之间十几纳秒的分发差。

### 8.4 三级缓存提前暴露代理的时机成本

- 提前暴露只发生在真正出现循环引用时：Bean 实例化后、属性填充前，
  容器把一个工厂注册进三级缓存 `singletonFactories`，注册本身只是
  放一个 lambda，成本可忽略；只有当另一个 Bean 在填充依赖时查到
  三级缓存，工厂才执行 `getEarlyBeanReference`，提前生成代理。
- 提前生成的代价等于一次代理创建：有循环且 Bean 带切面时，代理不在
  正常的 `postProcessAfterInitialization`（第 6 步）生成，而是提前到
  依赖填充阶段生成，成本约一次代理创建（见 8.1，微秒级），不是额外
  多付一次——容器用 `earlyProxyReferences` 记录已提前代理的 Bean，
  后置处理时不再重复代理。
- 二级缓存保证只生成一次：工厂执行后结果移入 `earlySingletonObjects`，
  工厂从三级缓存移除，后续依赖方直接命中二级缓存，不会重复生成代理，
  也不会出现同一个 Bean 两个代理实例。
- 无循环时三级缓存零成本：工厂注册后从未被触发，Bean 走正常流程在
  初始化后生成代理，三级缓存的工厂随 Bean 完成而丢弃，不产生代理、
  不产生额外类加载。

边界条件：

- 以上数字以单线程冷启动、JIT 未充分预热为前提；预热后分发与拦截
  开销都会下降，热路径应以 JMH 预热后的稳态值为准。
- 切面数量按拦截器个数线性叠加：3 个切面约 3 倍链开销，`@Around`
  里查库、写日志、调远程服务的成本不在本节数字内，须单独计量。
- CGLIB 生成类数量按被代理类型数算，不按 Bean 实例数算：1000 个
  同类 Bean 共用 1 个生成子类，1000 个不同类才生成 1000 个子类。
- 本节不适用于 GraalVM Native Image：AOT 下代理类在构建期生成，
  启动期无运行时生成成本，但构建期要显式登记代理，约束完全不同。

## 小技巧与冷知识

【小技巧】自调用想走代理有两个现成办法，不用把方法搬到别的类：自己注入自己（`@Autowired @Lazy private ApplyService self;`，调 `self.xxx()`）或开 `exposeProxy = true` 后用 `AopContext.currentProxy()`。前者靠 @Lazy 断环——自己依赖自己本来就是循环依赖，@Lazy 让容器先给一个代理占位；后者要在 `@EnableAspectJAutoProxy(exposeProxy = true)` 里显式打开，默认是关的，用完记得线程内取的是当前调用的代理。

【小技巧】`@Around` 里忘了调 `proceed()`，目标方法根本不会执行：Around 通知的返回值就是 `pjp.proceed()` 的返回值，漏写这一行等于把业务方法悄悄短路掉，接口返回 null、数据库没写、还不报错。写 Around 先写 `return pjp.proceed();` 再往外加计时、try-finally，别先写日志逻辑再补核心调用。

【小技巧】切面里吞掉异常会顺带吞掉事务回滚：Around 里 catch 住异常只记日志不重抛，调用方看到的是正常返回，事务拦截器也以为方法成功了，该回滚的数据就提交了。切面里要么不 catch、要么 catch 后处理完再 `throw e`；只想记日志就用 `@AfterThrowing`，别在 Around 里把异常消化掉。

【小技巧】`@Configuration` 的 `proxyBeanMethods = false` 是轻量模式：关掉后 `@Bean` 方法互调不再走代理、每次都是新 new 出来的对象，不再保证单例。原文第 5 节说默认 true 保证互调同一单例，反过来就是——配置类里 @Bean 方法之间没有互相调用时，关掉它能省启动时间和代理开销，Spring Boot 的自动配置类用的就是这个模式。

【小技巧】`@Autowired` 只在单构造器时可省：Spring 4.3 起，一个类只有一个构造器时不用标 `@Autowired`；有两个以上构造器时，必须显式标一个，告诉容器用哪一个，否则容器不知道选哪个注入。Lombok 的 `@RequiredArgsConstructor` 生成的也是单构造器，所以能直接省掉注解。

【小技巧】`BeanPostProcessor` 自己不能注入普通 Bean：它在容器启动早期就被实例化，比所有普通 Bean 都早，注入普通 Bean 会触发 `BeanCurrentlyInCreationException`。自定义后置处理器里需要别的 Bean 时，用 `ApplicationContextAware` 拿到容器、等用时再 `getBean`，别在字段上直接 `@Autowired`。

【小技巧】切入点里 `*` 和 `..` 的匹配层级不一样：`execution(* com.example.*.*(..))` 里包名位置的 `*` 只匹配一层包，`com.example.service` 能命中、`com.example.service.impl` 命中不了；想匹配所有子包要用 `..`，即 `execution(* com.example..*.*(..))`。切面只在部分包生效、子包里的类没被切到时，先看包名位置写的是 `*` 还是 `..`。

【小技巧】判断注入的是不是代理，看类名后缀就够了：JDK 代理的类名是 `$Proxy0` 这类，CGLIB 代理带 `$$EnhancerBySpringCGLIB$$`。更可靠的是 `AopUtils.isAopProxy(bean)`，调试“为什么这个 Bean 的行为和源码对不上”时，先确认拿到的是代理还是原始对象，再决定是去看切面还是去看业务代码。

【小技巧】构造器循环依赖用 `@Lazy` 能断环，但只是把环藏起来：`@Lazy` 让容器先注入一个代理占位，真正调用时才解析目标 Bean，启动不再报错。原文第 4 节建议拆环而不是藏环——@Lazy 适合临时过渡，长期留着等于把环依赖的爆炸点从启动期推迟到运行期，某个方法第一次被调时才暴露。

【冷知识】“控制反转”这个名字是 Martin Fowler 在 2004 年的文章《Inversion of Control Containers and the Dependency Injection Pattern》里定下来的，DI（依赖注入）也是他在同一篇文章里给出的叫法。Spring 的前身是 Rod Johnson 2002 年的 Interface21 框架，脱胎于他写的《Expert One-on-One J2EE Design and Development》一书里的示例代码。

【冷知识】Spring AOP 只支持方法执行这一个连接点：字段读写、构造器调用、静态初始化这些 AspectJ 能切的点，Spring AOP 都切不了——因为它只是运行时代理，只能在方法调用前后织入。需要切字段或构造器的场景，得用 AspectJ 的编译期或加载期织入，Spring AOP 的表达式再怎么写也够不着。

【冷知识】JDK 动态代理要求目标有接口、CGLIB 要求类和方法不能 final，根子在两者的生成方式：JDK 代理是运行时生成一个实现目标接口的新类，根本不继承目标类；CGLIB 是生成目标类的子类、重写方法做拦截，final 方法子类重写不了、final 类干脆没法继承。理解这一点就明白为什么“没有接口又想用 JDK 代理”从一开始就不成立。

## 9. 面试与自测

1. Bean 生命周期按顺序说一遍，AOP 代理在哪一步产生？
2. 三级缓存各自放什么？为什么需要第三级工厂而不是二级直接放代理？
3. JDK 代理与 CGLIB 的区别？Boot 2.7 默认用哪个？
4. 构造器循环依赖为什么解决不了？你会怎么改设计？
5. 事务失效与切面不生效的共同根源是什么？

6. 一个应用有 1000 个被代理 Bean，CGLIB 代理生成总共约多少毫秒？为什么说代理生成不是启动慢的主因？单次 Spring AOP 拦截开销约多少纳秒，什么情况下它才值得优化？

### 参考答案

1. 顺序是：实例化→依赖注入→Aware 回调→BeanPostProcessor 前置（含 @PostConstruct 处理）→初始化（afterPropertiesSet/init-method）→BeanPostProcessor 后置→使用与销毁。AOP 代理在第 6 步后置处理中产生，由 AbstractAutoProxyCreator 匹配切面后返回代理替换原 Bean。因此初始化回调里自调本类方法时代理尚未生成，切面逻辑不会生效。
2. 一级 singletonObjects 放成品 Bean；二级 earlySingletonObjects 放已实例化、未完成注入的半成品早期引用；三级 singletonFactories 放半成品工厂。不能由二级直接放代理，是因为只有真正发生循环引用、需要提前暴露时才应生成代理；工厂按需生成，既保证循环方拿到的是代理而非原始对象，又避免无循环的 Bean 都被提前代理。
3. JDK 动态代理生成实现目标接口的代理类，要求目标有接口；CGLIB 生成目标类的子类，要求类和方法不能为 final。Spring Boot 2.7 默认 proxyTargetClass=true，即默认用 CGLIB，不论目标是否有接口。
4. 构造器注入要求依赖在构造前已完全就绪，而循环的双方都等对方先构造完成，没有可暴露的半成品，所以无解，启动直接报错。这正是设计有环的信号，应拆环：把共用逻辑抽到第三个 Bean，或改事件解耦；不要换成字段注入或 @Lazy 把环藏起来。
5. 共同根源是绕过了 AOP 代理：事务本身就是建在 AOP 代理上的声明式能力。自调用（this.）、容器外 new 的对象、private/final 方法都不会经过代理拦截器，因此事务拦截器和切面都不会执行，表现为事务失效或切面不生效。

6. 约27.5毫秒，仅占启动约1%；大头是固定成本与每Bean约2.2毫秒创建注入。单次裸链拦截约275–500纳秒，带切面逻辑约1.5微秒，毫秒级方法可忽略。

## 10. 关联知识

- 前置：[Spring Boot](开发语言/Java/Spring/Spring-Boot.md)、[设计模式](开发语言/Java/设计模式.md)（代理模式）
- 相关：[Spring 事务](开发语言/Java/Spring/Spring-事务.md)（最重要的 AOP 落点）
- 延伸：AspectJ 编译期织入、Spring 事件机制解耦环依赖
