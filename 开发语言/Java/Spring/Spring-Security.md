---
知识点: Spring Security
层级: 进阶
适用版本: Spring Security 5.7 / Spring Boot 2.7
更新日期: 2026-11-14
关联:
  - Spring-Boot
  - 接口设计与文档
标签:
  - Java
  - 原理
  - 实践
---

# Spring Security

## 1. 核心概念

Spring Security 解决两件事：**认证**（你是谁）和**授权**（你能干什么）。它不在 Spring MVC 里面，而是以一条 Servlet 过滤器链的形式拦在 DispatcherServlet 之前——请求先进安全链，验完身份、判完权限，才轮到 Controller。

理解它就抓一条主线：过滤器链负责流程骨架，认证信息放在 `SecurityContext` 里随请求传递，授权决策只认这份上下文，不直接关心你怎么登录的。

## 2. 过滤器链机制

- Boot 应用里通过 `DelegatingFilterProxy` 把 Servlet 容器的过滤器桥接到 Spring 管理的安全链；一条请求可能走多条 `SecurityFilterChain`（按 URL 模式匹配，先命中先用），如 `/api/**` 一条（无状态、接口口径）、`/admin/**` 一条（表单登录口径）。
- 链上关键过滤器（按执行顺序的骨干，不是全量）：

| 过滤器 | 职责 |
|---|---|
| `SecurityContextPersistenceFilter` | 请求开始时从 Session 加载 SecurityContext，结束时存回 |
| `UsernamePasswordAuthenticationFilter` | 处理表单登录提交，组装待认证对象 |
| `ExceptionTranslationFilter` | 把认证/授权异常翻译成 401 响应或跳登录页 |
| `AuthorizationFilter`（5.7 新名，原 FilterSecurityInterceptor） | 链尾做 URL 授权判定，权限不够抛异常 |

- 调试口径：启动日志开 `logging.level.org.springframework.security=DEBUG` 能看到实际生效的链顺序；排查「配置没生效」先确认请求命中的是哪条链，而不是先怀疑表达式写错。

## 3. 认证流程

1. 登录请求被认证过滤器拦下，生成一个未认证的 `Authentication`（含用户名、密码凭证）。
2. 交给 `AuthenticationManager`（默认实现 `ProviderManager`），它逐个问链上的 `AuthenticationProvider` 谁能处理。
3. 表单登录走 `DaoAuthenticationProvider`：调 `UserDetailsService.loadUserByUsername` 查用户，用 `PasswordEncoder` 比对密码，成功后组装带权限集合（GrantedAuthority）的已认证对象。
4. 结果放进 `SecurityContextHolder`（内部是 ThreadLocal，所以异步线程里默认拿不到、要显式传递），后续授权与业务代码从这里取当前用户。

业务侧实现认证通常只需两件：自己的 `UserDetailsService`（从用户表查账号、状态、角色）和密码编码器 Bean，其余流程框架托管。

## 4. 授权与 Boot 2.7 的现行写法

5.7 起官方主推**组件式写法**：不再继承 `WebSecurityConfigurerAdapter`（已废弃，见第 8 节），改成暴露 `SecurityFilterChain` Bean：

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        http.antMatcher("/api/**")
            .csrf().disable()                       // 纯接口、无浏览器表单时才关
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests(auth -> auth     // 5.7 新 API，旧 authorizeRequests 也可用
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated());
        return http.build();
    }
}
```

- URL 级授权是粗筛：先按路径定大规则。细粒度到「这个角色能点这个按钮/调这个方法」用方法级安全：`@EnableMethodSecurity`（5.7 新注解）+ `@PreAuthorize("hasAuthority('loan:approve')")`，权限点用字符串常量集中管理，不要散落在各处手写。
- 角色与权限分开：`hasRole` 匹配的是 `ROLE_` 前缀的权限，业务系统里更实用的是按权限点（菜单、按钮、接口）授权，角色只是权限点的集合。

## 5. 密码存储

- 只用自适应哈希：`BCryptPasswordEncoder` 是默认选择，成本因子可调、每个密码自带盐。MD5、SHA-1 裸哈希、以及「自己发明的加盐规则」一律不合格。
- `DelegatingPasswordEncoder` 靠存储前缀分发：密文形如 `{bcrypt}$2a$10$...`，`{noop}` 是明文对比，只允许出现在本地调试，上生产前全局搜一遍清掉。
- 常见报错 `There is no PasswordEncoder mapped for the id "null"`：库里存的是无前缀的老密文，与委托编码器对不上。迁移路径是登录时校验老算法、成功后按新格式重写密文，逐步洗库，而不是把编码器改回兼容明文。

## 6. 与 JWT / OAuth2 的关系（概念与选型）

- **JWT**：自包含的签名令牌，服务端不存会话、验签即认证。优点是无状态易横向扩展；硬伤是签发后难以撤销（拉黑名单等于把状态又加回来）、令牌体积与信息泄露面（payload 只是 Base64，别放敏感字段）。
- **OAuth2**：是**授权框架**，解决「让第三方应用有限访问用户资源」，不是认证协议；登录语义靠在其之上的 OIDC 补齐。自家单系统没有第三方接入诉求时，上 OAuth2 属于杀鸡用牛刀。
- 选型口径（ao 的企业后台场景）：浏览器后台 + 单体/少量实例，用 Session + Spring Security 默认机制最稳，CSRF、会话固定防护都是现成的；纯接口给前端/App 调用、要无状态扩展，再用 JWT 且接受短有效期 + 刷新令牌的配套复杂度。ao 早年在文旅大数据项目里用的就是 Spring Security 做后台认证授权，这套过滤器链 + 权限点的打法到今天仍然成立，只是配置写法换成了组件式。

## 7. 疑难杂症

- **分不清 401 和 403**：没认证（或凭证无效）是 401，未认证访问受保护资源时由认证入口点处理；已认证但权限不够是 403，由 AccessDeniedHandler 处理。接口给前端时两者都要约定清（与 [REST 接口设计](工程实践/接口设计与文档.md) 的状态码纪律对齐），别统一包成 200+业务码让前端猜。
- **接口莫名被拦或莫名放行**：多条链时请求命中了非预期链（匹配顺序按 Bean 定义顺序）。先打 DEBUG 日志看「Securing GET /xxx」走的是哪条链。
- **CSRF 403 误杀接口**：浏览器表单链开了 CSRF，而 Postman/前端没带 token。纯接口链才关 CSRF；带浏览器 Cookie 会话的链不要关，关了等于拆掉一层真实防护。
- **异步/定时任务里取不到当前用户**：SecurityContext 在 ThreadLocal 上，换线程就丢。需要在提交任务前显式捕获并传递，或把所需用户信息作为参数传下去，不要指望框架自动跟过去。

## 8. 版本差异

- 5.7（Boot 2.7 对应线）：`WebSecurityConfigurerAdapter` 废弃，改 `SecurityFilterChain` Bean 组件式配置；`authorizeHttpRequests` 与 `AuthorizationManager` 体系引入，旧 `authorizeRequests` 仍可用。
- Boot 3 / Security 6：包名换 Jakarta、`antMatchers` 换 `requestMatchers`，写法主线延续 5.7 的组件式。本篇代码按 5.7 口径，升级时对照这两点改，不要把 6.x 的示例代码抄进 2.7 项目。

## 9. 认证成本定量：BCrypt 成本因子与过滤器链开销算一算

本节只算两笔账：一次登录里密码哈希烧掉多少时间，以及整条过滤器链摊到每次请求上是多少。结论先说在前面——登录慢，瓶颈几乎总在 BCrypt，不在过滤器链。

### 9.1 成本因子是指数，不是线性

BCrypt 的成本因子 `cost` 是指数：内部轮数是 `2^cost`。`cost` 每加 1，轮数翻一倍，哈希耗时也大约翻一倍；`cost` 加 2，耗时翻两番。

| 成本因子 | 轮数（2^cost） | 单次哈希耗时（估算） | 单核每秒可验 |
|---|---|---|---|
| 10 | 1024 | 约 62 ms | 约 16.1 次 |
| 11 | 2048 | 约 125 ms | 约 8.0 次 |
| 12 | 4096 | 约 249 ms | 约 4.0 次 |
| 13 | 8192 | 约 498 ms | 约 2.0 次 |

表中毫秒数取自 AWS c7i.xlarge（Ubuntu 24.04）上的一组公开实测，是估算口径：换一台机器数字会变，但“每加 1 翻一倍”的比例关系不变。`new BCryptPasswordEncoder()` 的默认成本因子就是 10。

【算例 1】把成本因子从 10 调到 12，单次哈希从约 62 ms 涨到约 249 ms：249 ÷ 62 ≈ 4.0 倍。原文小技巧里“从 10 调到 12，单次登录的 CPU 开销翻两番”就是这笔账：2^2 = 4。再调到 13 则是 498 ÷ 62 ≈ 8.0 倍。

### 9.2 OWASP 建议口径

OWASP Password Storage Cheat Sheet 对 bcrypt 的口径是：

- bcrypt 只用于 Argon2 和 scrypt 都不可用的遗留系统，新系统首选 Argon2id；
- 成本因子最低 10，越大越好，以验证服务器性能允许为限；
- 单次哈希耗时一般应小于 1 秒，超过就该降成本或加机器；
- 密码最长 72 字节，超长部分多数实现会截断。

按 9.1 的实测锚点换算：cost 12 约 249 ms，落在“小于 1 秒”的口径内，是多数生产系统的常用值；cost 13 约 498 ms，仍在口径内，但已吃掉半秒登录预算；cost 14 按翻倍推算约 1 秒，已顶到口径上限，一般不建议直接上。

### 9.3 密码哈希占登录总耗时的比例

一次登录的耗时拆成三块：查用户（DB 一次点查，约 5 ms）+ 密码哈希（BCrypt 校验）+ 其余（会话、响应，约 10 ms）。

【算例 2】cost 10 时：62 + 5 + 10 = 77 ms，哈希占 62 ÷ 77 ≈ 80.5%。cost 12 时：249 + 5 + 10 = 264 ms，哈希占 249 ÷ 264 ≈ 94.3%。

也就是说，cost 12 的登录请求里，超过九成时间都在等 BCrypt。DB 查用户那 5 ms 再怎么优化，对登录总耗时也几乎没有影响；想让登录变快，只能动成本因子、加 CPU，或者把校验拆到独立线程池。

### 9.4 过滤器链的请求开销

默认配置下，一条 `SecurityFilterChain` 约有 15 个过滤器（表单登录 + HTTP Basic 的默认组合，5.7 口径下以 `SecurityContextPersistenceFilter` 开头、`AuthorizationFilter` 收尾）。单个过滤器只做一次判断或一次上下文读写，没有循环、没有 I/O，耗时是微秒级。

【算例 3】按单个过滤器约 10 μs 估算：15 × 10 μs = 150 μs = 0.15 ms。按每过滤器 5–20 μs 的区间算，总开销在 0.07–0.30 ms 之间。

对比：0.15 ÷ 249 ≈ 0.06%（相对 cost 12 的一次哈希）；0.15 ÷ 50 ≈ 0.3%（相对一次 50 ms 的业务处理）。过滤器链的开销连一次哈希的零头都不到，更不到业务耗时的 1%。

所以“链太长导致接口慢”基本是个错觉：真要优化登录延迟，先看 BCrypt 的成本因子，再看 DB 和业务；过滤器链只有在自定义过滤器里写了重逻辑（比如每次请求调外部接口）时才值得怀疑。

### 9.5 并发登录下的 CPU 换算

BCrypt 校验是纯 CPU 计算，不占内存、不等 I/O，多少次并发登录就需要多少核。单核每秒能验几次，9.1 的表里已经算好：cost 12 是约 4.0 次/秒。

【算例 4】cost 12、单次 249 ms 时：1 次登录/秒占 0.25 核，5 次/秒占 1.25 核，10 次/秒占约 2.5 核。8 核机器的理论上限是 8 ÷ 0.249 ≈ 32.1 次登录/秒；同样 8 核，cost 10 时上限约 129 次/秒。1 分钟涌入 1000 次登录：cost 12 耗 CPU 约 1000 × 0.249 = 249 秒 ≈ 4.15 核·分钟，cost 10 只要约 62 秒。

换算成容量规划：登录峰值按“每秒登录次数 × 单次哈希秒数”直接折成核数，再留 30% 余量。超过核数时请求不会报错，只会排队——登录接口的 P95 延迟会被 BCrypt 排队拖垮，这是登录高峰期最常见的表象。

### 9.6 边界条件

- 本节毫秒数是估算口径，锚点是一台 AWS c7i.xlarge 的公开实测；生产值以实测为准，调成本因子前先在生产同款硬件上跑一遍压测。
- 成本因子每加 1 耗时翻倍是比例关系，任何机器都成立；但绝对毫秒数随 CPU 型号、JVM、并发度而变，不能直接把本节数字抄进容量文档。
- 哈希耗时是 CPU 耗时，阻塞的是执行校验的那个线程；Tomcat 线程池被 BCrypt 占满时，不登录的请求也会跟着排队。
- cost 12 以上时，登录接口必须配限流；攻击者用错误密码反复登录，烧的是服务器 CPU，成本因子越高，DoS 效果越好。

### 9.7 选型口径

- 默认用 `new BCryptPasswordEncoder()`（cost 10），确认压测通过后再上调到 12，不要一上来就写 12。
- 登录接口的 P95 延迟预算里，给 cost 12 预留约 250 ms 的哈希时间；预算不够就降成本，别让登录超时先爆。
- 高峰期扛不住时先限流、后降成本：降成本是安全降级，要走变更评审，不能悄悄改一行配置就上线。

## 原理剖析

本节只把原文已有章节给出的结论拆开讲清机制：不引入新概念，不补新数字，数字一律复用 §9 已核验的定量算例；凡原文没展开的点，文中明确标注，不作定论。

### 过滤器链的执行顺序与放行机制（§2）

请求先进安全链、再到 Controller，这条主线在 §1 已定下，§2 讲的是链本身怎么组织。Boot 应用里，Servlet 容器只认一个过滤器入口，`DelegatingFilterProxy` 把它桥接到 Spring 管理的安全链上——容器与 Spring 各管一层，桥接是两者之间的接头。原文未展开桥接之外的细节，例如代理如何按 Bean 名查找目标链。

一条请求可能走多条 `SecurityFilterChain`，按 URL 模式匹配，先命中先用。原文的例子是 `/api/**` 一条（无状态、接口口径）、`/admin/**` 一条（表单登录口径）：匹配上的那条链全程负责，没命中的链不参与。这就是“放行”的第一层含义——不是某个过滤器放行，而是请求一开始就被分流到某一条链；配错了 URL 模式，请求进的就不是你以为的那条链。§7 的“接口莫名被拦或莫名放行”正是这个机制的反面：多条链时请求命中了非预期链，匹配顺序按 Bean 定义顺序。

链内的执行顺序是 §2 表格给出的骨干顺序：`SecurityContextPersistenceFilter` 打头，请求开始时从 Session 加载 SecurityContext、结束时存回；`UsernamePasswordAuthenticationFilter` 在中间处理表单登录提交；`ExceptionTranslationFilter` 负责把认证/授权异常翻译成 401 响应或跳登录页；链尾的 `AuthorizationFilter`（5.7 新名，原 `FilterSecurityInterceptor`）做 URL 授权判定，权限不够抛异常，再由前面的 `ExceptionTranslationFilter` 翻译成前端看到的状态码。原文明确标注这只是“骨干，不是全量”，完整的过滤器清单原文未展开。

所以“放行”分三层：第一层是链的选择（先命中先用），第二层是链内每个过滤器按顺序各做一次判断或读写、通过就往下走，第三层是链尾授权判定通过才进 Controller。调试口径也在 §2 给出：启动日志开 `logging.level.org.springframework.security=DEBUG` 能看到实际生效的链顺序；排查“配置没生效”先确认请求命中的是哪条链，而不是先怀疑表达式写错。

### 认证流程中 AuthenticationManager 与认证信息的流转（§3）

§3 的认证流程是四步接力，认证信息每一步换一个形态，但始终是一份 `Authentication` 对象在流转。

第一步，登录请求被认证过滤器拦下，生成一个未认证的 `Authentication`（含用户名、密码凭证）。注意这时它还只是“待认证”——用户名密码都在，但没人验过。

第二步，这份未认证对象交给 `AuthenticationManager`（默认实现 `ProviderManager`），它逐个问链上的 `AuthenticationProvider` 谁能处理。原文未展开“逐个问”的失败处理细节，例如没有一个 Provider 能处理时框架如何收尾，不作定论。

第三步，表单登录走 `DaoAuthenticationProvider`：调 `UserDetailsService.loadUserByUsername` 查用户，用 `PasswordEncoder` 比对密码，成功后组装带权限集合（`GrantedAuthority`）的已认证对象。查用户、比对密码、组装权限是三个不同的职责，分属三个组件：`UserDetailsService` 只管查账号、状态、角色，`PasswordEncoder` 只管比对，Provider 只管把结果拼成已认证对象。

第四步，结果放进 `SecurityContextHolder`（内部是 ThreadLocal），后续授权与业务代码从这里取当前用户。ThreadLocal 决定了这份认证信息的生命周期：它随请求线程传递，换线程就丢——§7 的“异步/定时任务里取不到当前用户”就是这个机制的直接后果，需要在提交任务前显式捕获并传递，或把所需用户信息作为参数传下去。

业务侧实现认证通常只需两件（§3 原文）：自己的 `UserDetailsService` 和密码编码器 Bean，其余流程框架托管。也就是说，认证流程的骨架是固定的，业务只在“查用户”和“编密码”两个点上插自己的实现。

### 授权判定与 Boot 2.7 现行写法的取舍（§4）

授权判定只认 `SecurityContext` 里的已认证对象，不直接关心你怎么登录的（§1 的主线）。判定分两层：URL 级是粗筛，先按路径定大规则；方法级是细筛，到“这个角色能点这个按钮/调这个方法”。

5.7 起官方主推组件式写法（§4、§8）：不再继承 `WebSecurityConfigurerAdapter`（已废弃，见 §8），改成暴露 `SecurityFilterChain` Bean。原文 §4 的示例只给了一种链的写法：

```java
@Bean
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http.antMatcher("/api/**")
        .csrf().disable()
        .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .and()
        .authorizeHttpRequests(auth -> auth
            .antMatchers("/api/public/**").permitAll()
            .antMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated());
    return http.build();
}
```

这段配置同时体现了三处取舍。第一，`antMatcher("/api/**")` 定的是这条链的管辖范围，与 §2“先命中先用”的多链分流是同一机制。第二，`csrf().disable()` 只在纯接口、无浏览器表单时才关（§4 原文注释）；带浏览器 Cookie 会话的链不要关，§7 也提醒关了等于拆掉一层真实防护。第三，`sessionCreationPolicy(SessionCreationPolicy.STATELESS)` 配的是无状态口径，与 §2 例子里 `/api/**` 无状态、`admin/**` 表单登录的对照一致。

授权表达式的取舍在 §4 写明：`hasRole` 匹配的是 `ROLE_` 前缀的权限，业务系统里更实用的是按权限点（菜单、按钮、接口）授权，角色只是权限点的集合。细粒度授权用方法级安全：`@EnableMethodSecurity`（5.7 新注解）+ `@PreAuthorize("hasAuthority('loan:approve')")`，权限点用字符串常量集中管理，不要散落在各处手写。`AuthorizationFilter` 在链尾做 URL 授权判定的内部细节原文未展开，不作定论。

### 密码存储为什么用 BCrypt（§5，结合 §9）

§5 的结论是“只用自适应哈希：`BCryptPasswordEncoder` 是默认选择”。“自适应”体现在两点上，一是成本因子可调，二是每个密码自带盐。MD5、SHA-1 裸哈希、以及“自己发明的加盐规则”一律不合格——原文只给了结论，没有展开裸哈希为什么不合格的攻击细节，原文未展开，不作定论。

成本因子可调的意义要结合 §9 的定量算例看。BCrypt 的成本因子 `cost` 是指数：内部轮数是 `2^cost`，`cost` 每加 1，哈希耗时大约翻一倍（§9.1）。§9.1 的实测锚点是 AWS c7i.xlarge（Ubuntu 24.04）上的一组公开实测：cost 10 单次哈希约 62 ms，cost 11 约 125 ms，cost 12 约 249 ms，cost 13 约 498 ms；`new BCryptPasswordEncoder()` 的默认成本因子就是 10。把成本因子从 10 调到 12，单次哈希从约 62 ms 涨到约 249 ms，249 ÷ 62 ≈ 4.0 倍，即 2^2 = 4；再调到 13 则是 498 ÷ 62 ≈ 8.0 倍。所谓“自适应”，就是硬件变快了可以把成本因子往上调，让哈希耗时重新回到可接受的区间，而不是像固定轮数的哈希那样越跑越快、越跑越不安全。

存储格式上，`DelegatingPasswordEncoder` 靠存储前缀分发：密文形如 `{bcrypt}$2a$10$...`，`{noop}` 是明文对比，只允许出现在本地调试，上生产前全局搜一遍清掉（§5）。前缀解决的是迁移问题：新旧算法的密文可以共存，登录时校验老算法、成功后按新格式重写密文，逐步洗库，而不是把编码器改回兼容明文。常见报错 `There is no PasswordEncoder mapped for the id "null"` 就是库里存了无前缀的老密文、与委托编码器对不上的表现（§5）。BCrypt 内部轮数之外的算法细节原文未展开，不作定论。

## 实战案例

### 案例一：登录成功却 403，未登录却跳登录页

现象：用户能正常登录，登录成功后访问 `/api/admin/**` 下的接口却返回 403；另一组用户未登录直接访问受保护接口，浏览器跳登录页、接口调用方拿到的是跳转而不是 401。两个现象看似相反，排查用的是同一套流程（§3、§4）。

排查：第一步先确认请求命中的是哪条链。按 §2 的调试口径开 `logging.level.org.springframework.security=DEBUG`，看日志里“Securing GET /xxx”走的是哪条链——§7 提醒，多条链时请求命中非预期链是最常见的“莫名被拦或莫名放行”，匹配顺序按 Bean 定义顺序。如果请求根本没进预期的 `apiChain`，问题在链的 URL 匹配，不在授权表达式。

第二步，确认链对了之后，再沿 §3 的认证流转查：登录是否真的走完了四步。登录请求被认证过滤器拦下生成未认证 `Authentication`，经 `AuthenticationManager`（默认 `ProviderManager`）交给 `DaoAuthenticationProvider`，调 `UserDetailsService.loadUserByUsername` 查用户、`PasswordEncoder` 比对密码，成功后带权限集合的已认证对象放进 `SecurityContextHolder`。如果登录成功但权限集合里没有 `ROLE_ADMIN`，链尾 `AuthorizationFilter` 按 `.antMatchers("/api/admin/**").hasRole("ADMIN")`（§4 示例）判定必然是 403——登录成功只证明认证过了，不证明权限够了。

第三步，对照 §4 的角色与权限分开的写法查表达式：`hasRole` 匹配的是 `ROLE_` 前缀的权限。如果权限点存的是 `loan:approve` 这种字符串、或角色存的是 `ADMIN` 却用了 `hasAuthority("ADMIN")` 原样比对，表达式与权限集合对不上，同样是 403。方法级 `@PreAuthorize("hasAuthority('loan:approve')")` 不生效时，先查 §4 的开关 `@EnableMethodSecurity` 有没有开——原文 §4 只给了注解名与用法组合，开关缺失时的具体静默表现原文未展开，只按 §4 的写法核对开关是否存在。

第四步，未登录跳登录页的那一半是另一条分支：§7 写明，没认证（或凭证无效）是 401，未认证访问受保护资源时由认证入口点处理；已认证但权限不够是 403，由 AccessDeniedHandler 处理。两者在 `ExceptionTranslationFilter`（§2）里被翻译成不同的响应：浏览器表单链跳登录页是符合 §2 表格描述的正常行为，接口链则应按 §7 的口径与 [REST 接口设计](工程实践/接口设计与文档.md) 的状态码纪律对齐，给前端约定的 401，而不是让前端猜跳转。

结论：登录成功却 403，优先按“链是否命中 → 权限集合里有没有 `ROLE_ADMIN` → 表达式是 `hasRole` 还是 `hasAuthority`”三步查，不要先怀疑登录本身；未登录跳登录页是认证入口点的正常翻译，与 403 的 AccessDeniedHandler 是两条分支，接口返回什么状态码要按 §7 的 401/403 区分约定，而不是把两者混成一种现象处理。

### 案例二：成本因子从 10 调到 12，登录变慢了

现象：上线时把 `BCryptPasswordEncoder` 的成本因子从默认 10 调到 12，登录接口 P95 明显变慢，有人怀疑是过滤器链太长拖慢了接口。按 §9 的三笔账定位。

排查：第一步算哈希本身。§9.1 的表：cost 10 单次哈希约 62 ms，cost 12 约 249 ms，249 ÷ 62 ≈ 4.0 倍，即成本因子每加 1 耗时翻倍、加 2 翻两番（2^2 = 4）。再调到 13 是约 498 ms，498 ÷ 62 ≈ 8.0 倍。

第二步算哈希在登录总耗时里的占比。§9.3 的拆分是查用户（DB 一次点查，约 5 ms）+ 密码哈希 + 其余（会话、响应，约 10 ms）。【算例 2】cost 10 时总耗时 62 + 5 + 10 = 77 ms，哈希占 62 ÷ 77 ≈ 80.5%；cost 12 时总耗时 249 + 5 + 10 = 264 ms，哈希占 249 ÷ 264 ≈ 94.3%。登录慢了约 264 − 77 = 187 ms，其中哈希一项就涨了约 249 − 62 = 187 ms——变慢的全部增量都在 BCrypt 上，DB 查用户那 5 ms 再怎么优化也几乎没有影响。

第三步算过滤器链的开销，验证“链太长”的怀疑。§9.4：默认配置下一条 `SecurityFilterChain` 约有 15 个过滤器，单个过滤器只做一次判断或一次上下文读写，没有循环、没有 I/O，耗时是微秒级。【算例 3】按单个过滤器约 10 μs 估算，15 × 10 μs = 150 μs = 0.15 ms，按每过滤器 5–20 μs 的区间算总开销在 0.07–0.30 ms 之间。对比一次哈希：0.15 ÷ 249 ≈ 0.06%；相对一次 50 ms 的业务处理是 0.15 ÷ 50 ≈ 0.3%。过滤器链的开销连一次哈希的零头都不到，更不到业务耗时的 1%。

第四步看并发容量，确认高峰期会不会更糟。§9.5：单核每秒可验次数 cost 12 是约 4.0 次/秒，cost 10 是约 16.1 次/秒。【算例 4】cost 12、单次 249 ms 时，1 次登录/秒占 0.25 核，5 次/秒占 1.25 核，10 次/秒占约 2.5 核；8 核机器的理论上限是 8 ÷ 0.249 ≈ 32.1 次登录/秒，同样 8 核、cost 10 时上限约 129 次/秒。1 分钟涌入 1000 次登录：cost 12 耗 CPU 约 1000 × 0.249 = 249 秒 ≈ 4.15 核·分钟，cost 10 只要约 62 秒。超过核数时请求不会报错，只会排队，登录接口的 P95 延迟会被 BCrypt 排队拖垮。

结论：登录变慢的瓶颈在 BCrypt，不在过滤器链——cost 12 的一次登录里 94.3% 的时间都在等哈希，15 个过滤器只占 0.15 ms、约 0.06% 的哈希耗时。按 §9.7 的选型口径：默认用 `new BCryptPasswordEncoder()`（cost 10），确认压测通过后再上调到 12，不要一上来就写 12；登录接口的 P95 延迟预算里给 cost 12 预留约 250 ms 的哈希时间；高峰期扛不住时先限流、后降成本，降成本是安全降级，要走变更评审，不能悄悄改一行配置就上线。§9.6 也提醒：本节毫秒数是估算口径，锚点是一台 AWS c7i.xlarge 的公开实测，生产值以实测为准，调成本因子前先在生产同款硬件上跑一遍压测。

## 小技巧与冷知识

【小技巧】`hasRole("ADMIN")` 和 `hasAuthority("ADMIN")` 不是一回事：`hasRole` 会自动给角色名补 `ROLE_` 前缀，实际匹配的是 `ROLE_ADMIN` 这个权限；`hasAuthority` 是原样比对，不补前缀。权限点存的是 `loan:approve` 这种字符串时用 `hasAuthority`，角色存的是 `ADMIN` 时用 `hasRole`，两者混用是最常见的“明明配了权限却 403”的原因。

【小技巧】`permitAll()` 不等于跳过安全链：它只是“这条路不需要认证”，请求照样走完整条过滤器链，SecurityContext、异常翻译这些过滤器都在跑；`WebSecurity.ignoring()` 才是彻底跳过安全链，连过滤器都不进。静态资源、健康检查接口用 ignoring 更省事，但放行接口别图省事用它——一旦跳过链，连 CSRF、会话这些防护也一起没了。

【小技巧】方法级注解不生效先查开关有没有开：`@PreAuthorize`、`@PostAuthorize` 需要 `@EnableMethodSecurity`（5.7 新注解）才会生效，忘了加它，注解就静静地躺在方法上、一点作用不起，接口照常放行。老项目升级时也别忘了换注解名：5.7 之前是 `@EnableGlobalMethodSecurity(prePostEnabled = true)`，新注解默认就开了 pre/post。

【小技巧】BCrypt 的成本因子别盲目调高：`new BCryptPasswordEncoder()` 默认强度是 10（即 2 的 10 次方轮），强度每加 1，哈希耗时大约翻一倍。登录高峰期把强度从 10 调到 12，单次登录的 CPU 开销就翻两番；调之前先在登录接口上压测，确认耗时和吞吐都扛得住，别只在本地测一次就上线。

【小技巧】登录成功后会话 ID 会换：默认的会话固定防护是 `migrateSession`，认证成功时生成一个新 session id、把旧 session 的属性迁移过来，旧 id 立即作废。排障时看到“登录前后 session id 变了”不是 bug，是防会话固定攻击的正常行为；只有显式配了 `sessionFixation().none()` 才会保持原 id，生产别这么配。

【小技巧】想在 Controller 里直接拿当前用户，别自己去翻 SecurityContext：方法参数上写 `@AuthenticationPrincipal UserDetails user`，框架会把当前认证主体直接注入进来，比 `SecurityContextHolder.getContext().getAuthentication().getPrincipal()` 再强转干净得多。拿自定义的 User 对象时也可以这么写，注入的就是你 UserDetailsService 返回的那个类型。

【小技巧】“记住我”有两种实现，安全等级不一样：默认的基于哈希的 remember-me 只靠用户名、密码和密钥算签名，密码一改所有 remember-me 令牌立刻失效；持久化令牌版（`PersistentTokenRepository`）把 series/token 存进 `persistent_logins` 表，可以单独吊销某台设备的令牌。长期登录场景用持久化版，别只在配置里打开默认的 remember-me 就交差。

【小技巧】密码迁移可以“登录时自动升级”：`DelegatingPasswordEncoder` 配合 `UserDetailsPasswordService`，登录成功后如果发现库里存的是老算法密文，会自动按新编码器重算并写回库。原文第 5 节说的“登录时校验老算法、成功后重写密文逐步洗库”就是这套机制的落地形态，不用写批量洗库脚本，靠用户自然登录就能慢慢洗完。

【小技巧】登出默认只接受 POST：`logout()` 默认的登出地址是 POST `/logout`，并且要带 CSRF token，GET 请求直接访问会被 405 或 403 拦掉。前端登出按钮别写成 `<a href="/logout">`，要走表单 POST 或在请求头里带 token；只在纯接口、无浏览器会话的链上才考虑放开 GET 登出。

【冷知识】Spring Security 的前身叫 Acegi Security：2003 年由 Ben Alex 发起，2008 年随 2.0 版本正式改名 Spring Security 并并入 Spring 体系。所以老资料里搜 "Acegi" 能找到它最早期的过滤器链设计，思路和今天这套链式拦截一脉相承，只是名字和包名换了。

【冷知识】默认登录页不是你写的页面，是框架生成的：没有配自定义登录页时，访问 `/login` 看到的那个表单是 `DefaultLoginPageGeneratingFilter` 在运行期拼出来的 HTML。想改样式要么配自己的登录页，要么接管这个过滤器；直接改它生成的页面源码是改不动的，因为它根本不在你的代码里。

【冷知识】BCrypt 密文里的 `$2a$10$` 本身就是参数：`$2a$` 是算法版本，`10` 是成本因子，后面 22 个字符是盐，剩下的才是哈希值。所以只看密文开头就能知道这条密码用的是哪个版本、多大强度；`{bcrypt}$2a$10$...` 里花括号外是 DelegatingPasswordEncoder 的分发前缀，花括号内才是 BCrypt 自己的格式，两层含义别混在一起读。

## 10. 面试与自测

- 请求进 Controller 之前，Spring Security 在哪一层拦截？链的骨干过滤器各管什么？
- 认证成功的信息存在哪？为什么异步线程里取不到？
- DelegatingPasswordEncoder 的 `{bcrypt}` 前缀解决什么迁移问题？
- JWT 为什么难撤销？短令牌 + 刷新令牌是怎么折中的？
- 5.7 为什么废弃 WebSecurityConfigurerAdapter？

- 登录变慢了，先怀疑 BCrypt 还是过滤器链？cost 12 的 8 核机器能扛多少并发登录，超了会怎样？

### 参考答案

1. 在 Servlet 过滤器链层拦截，位于 DispatcherServlet 之前，经 DelegatingFilterProxy 桥接到 Spring 管理的安全链。骨干过滤器：SecurityContextPersistenceFilter 在请求开始加载、结束存回 SecurityContext；UsernamePasswordAuthenticationFilter 处理表单登录提交；ExceptionTranslationFilter 把认证/授权异常翻译成 401 或跳转登录页；链尾的 AuthorizationFilter 做 URL 授权判定。
2. 认证成功的信息存在 SecurityContextHolder 的 SecurityContext 里，其内部是 ThreadLocal，随请求线程传递。异步线程是新线程，默认没有这份 ThreadLocal 上下文，所以取不到当前用户，需在提交任务前显式捕获传递，或把所需用户信息作为参数传下去。
3. 前缀标识密文所用的编码器，如 {bcrypt}、{noop}，DelegatingPasswordEncoder 靠前缀分发到对应编码器比对。这样新旧算法的密文可以共存，迁移时登录先按老算法校验、成功后按新格式重写密文逐步洗库，而不必把编码器改回兼容明文。无前缀的老密文对不上委托编码器，就会报 no PasswordEncoder mapped for id "null"。
4. JWT 自包含、服务端不存会话，验签即认证，签发后在有效期内无法直接作废；要拉黑名单就等于把状态又加了回来，失去了无状态的意义。折中是短有效期访问令牌 + 刷新令牌：访问令牌过期快、泄露窗口小，刷新令牌可控、可撤销，配套复杂度是必须接受的代价。
5. 5.7 起官方主推组件式写法：暴露 SecurityFilterChain Bean，不再继承 WebSecurityConfigurerAdapter。继承式配置难以灵活组合多条链（不同 URL 用不同口径），组件式可按 URL 定义多条链、与 authorizeHttpRequests/AuthorizationManager 体系配合，配置更清晰、可组合。

6. 先看 BCrypt：cost 12 哈希约 249 ms、占登录 94%，cost 10 约 62 ms、占 80%；8 核机 cost 12 上限约 32 次/秒，10 次/秒占 2.5 核，超了会排队；15 个过滤器约 0.15 ms，仅为哈希的 0.06%，调成本前先实测。

## 11. 关联知识

### 前置知识

- [Spring-Boot](开发语言/Java/Spring/Spring-Boot.md)：本篇适用版本是 Spring Security 5.7 / Spring Boot 2.7（§11 已列为关联知识），Boot 的自动配置与项目基线是理解 §2 安全链如何被装配的前提。
- [Spring-MVC基础](开发语言/Java/Spring/Spring-MVC基础.md)：§1 讲安全链拦在 DispatcherServlet 之前，§2 的 DelegatingFilterProxy 与链内顺序都建立在 MVC 请求入口分层之上。
- [Spring-IOC与AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)：§4 的组件式写法本质是暴露 SecurityFilterChain 与 PasswordEncoder 两个 Bean，理解 Bean 装配才能读懂 §4 的配置取舍。
- [ThreadLocal](开发语言/Java/并发/ThreadLocal.md)：§3 的 SecurityContextHolder 内部是 ThreadLocal，§7 的异步线程取不到当前用户正是 ThreadLocal 换线程就丢的直接后果。
- [REST 接口设计](工程实践/接口设计与文档.md)：§7 的 401 与 403 区分要与该篇的状态码纪律对齐，接口给前端时两者都要约定清（§11 已列为关联知识）。
- [国密与加密体系](工程实践/国密与加密体系.md)：§5 的 BCrypt 自适应哈希与密码存储口径，与该篇的加密算法选型与敏感字段落库加密落地相关（§11 已列为关联知识）。

- [Spring-Boot](开发语言/Java/Spring/Spring-Boot.md)：自动配置与项目基线。
- [REST 接口设计](工程实践/接口设计与文档.md)：401/403 与统一错误口径的设计侧约定。

- [国密与加密体系](工程实践/国密与加密体系.md)：国密算法与银行敏感字段落库加密落地。
