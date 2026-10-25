---
知识点: Spring IOC 与 AOP
层级: 进阶
适用版本: Spring Boot 2.7.3 / Spring Framework 5.3
更新日期: 2026-10-23
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
- **接口耗时与慢调用告警**：`@Around` 计时，超阈值打 WARN，与 [监控告警](工程实践/监控告警.md) 的 Timer 埋点二选一，不要两头都做。
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

## 8. 面试与自测

1. Bean 生命周期按顺序说一遍，AOP 代理在哪一步产生？
2. 三级缓存各自放什么？为什么需要第三级工厂而不是二级直接放代理？
3. JDK 代理与 CGLIB 的区别？Boot 2.7 默认用哪个？
4. 构造器循环依赖为什么解决不了？你会怎么改设计？
5. 事务失效与切面不生效的共同根源是什么？

### 参考答案

1. 顺序是：实例化→依赖注入→Aware 回调→BeanPostProcessor 前置（含 @PostConstruct 处理）→初始化（afterPropertiesSet/init-method）→BeanPostProcessor 后置→使用与销毁。AOP 代理在第 6 步后置处理中产生，由 AbstractAutoProxyCreator 匹配切面后返回代理替换原 Bean。因此初始化回调里自调本类方法时代理尚未生成，切面逻辑不会生效。
2. 一级 singletonObjects 放成品 Bean；二级 earlySingletonObjects 放已实例化、未完成注入的半成品早期引用；三级 singletonFactories 放半成品工厂。不能由二级直接放代理，是因为只有真正发生循环引用、需要提前暴露时才应生成代理；工厂按需生成，既保证循环方拿到的是代理而非原始对象，又避免无循环的 Bean 都被提前代理。
3. JDK 动态代理生成实现目标接口的代理类，要求目标有接口；CGLIB 生成目标类的子类，要求类和方法不能为 final。Spring Boot 2.7 默认 proxyTargetClass=true，即默认用 CGLIB，不论目标是否有接口。
4. 构造器注入要求依赖在构造前已完全就绪，而循环的双方都等对方先构造完成，没有可暴露的半成品，所以无解，启动直接报错。这正是设计有环的信号，应拆环：把共用逻辑抽到第三个 Bean，或改事件解耦；不要换成字段注入或 @Lazy 把环藏起来。
5. 共同根源是绕过了 AOP 代理：事务本身就是建在 AOP 代理上的声明式能力。自调用（this.）、容器外 new 的对象、private/final 方法都不会经过代理拦截器，因此事务拦截器和切面都不会执行，表现为事务失效或切面不生效。

## 9. 关联知识

- 前置：[Spring Boot](开发语言/Java/Spring/Spring-Boot.md)、[设计模式](开发语言/Java/设计模式.md)（代理模式）
- 相关：[Spring 事务](开发语言/Java/Spring/Spring-事务.md)（最重要的 AOP 落点）
- 延伸：AspectJ 编译期织入、Spring 事件机制解耦环依赖
