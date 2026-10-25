---
知识点: Spring Security
层级: 进阶
适用版本: Spring Security 5.7 / Spring Boot 2.7
更新日期: 2026-10-23
关联:
  - Spring-Boot
  - REST 接口设计
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

- **分不清 401 和 403**：没认证（或凭证无效）是 401，未认证访问受保护资源时由认证入口点处理；已认证但权限不够是 403，由 AccessDeniedHandler 处理。接口给前端时两者都要约定清（与 [REST 接口设计](工程实践/REST接口设计.md) 的状态码纪律对齐），别统一包成 200+业务码让前端猜。
- **接口莫名被拦或莫名放行**：多条链时请求命中了非预期链（匹配顺序按 Bean 定义顺序）。先打 DEBUG 日志看「Securing GET /xxx」走的是哪条链。
- **CSRF 403 误杀接口**：浏览器表单链开了 CSRF，而 Postman/前端没带 token。纯接口链才关 CSRF；带浏览器 Cookie 会话的链不要关，关了等于拆掉一层真实防护。
- **异步/定时任务里取不到当前用户**：SecurityContext 在 ThreadLocal 上，换线程就丢。需要在提交任务前显式捕获并传递，或把所需用户信息作为参数传下去，不要指望框架自动跟过去。

## 8. 版本差异

- 5.7（Boot 2.7 对应线）：`WebSecurityConfigurerAdapter` 废弃，改 `SecurityFilterChain` Bean 组件式配置；`authorizeHttpRequests` 与 `AuthorizationManager` 体系引入，旧 `authorizeRequests` 仍可用。
- Boot 3 / Security 6：包名换 Jakarta、`antMatchers` 换 `requestMatchers`，写法主线延续 5.7 的组件式。本篇代码按 5.7 口径，升级时对照这两点改，不要把 6.x 的示例代码抄进 2.7 项目。

## 9. 面试与自测

- 请求进 Controller 之前，Spring Security 在哪一层拦截？链的骨干过滤器各管什么？
- 认证成功的信息存在哪？为什么异步线程里取不到？
- DelegatingPasswordEncoder 的 `{bcrypt}` 前缀解决什么迁移问题？
- JWT 为什么难撤销？短令牌 + 刷新令牌是怎么折中的？
- 5.7 为什么废弃 WebSecurityConfigurerAdapter？

### 参考答案

1. 在 Servlet 过滤器链层拦截，位于 DispatcherServlet 之前，经 DelegatingFilterProxy 桥接到 Spring 管理的安全链。骨干过滤器：SecurityContextPersistenceFilter 在请求开始加载、结束存回 SecurityContext；UsernamePasswordAuthenticationFilter 处理表单登录提交；ExceptionTranslationFilter 把认证/授权异常翻译成 401 或跳转登录页；链尾的 AuthorizationFilter 做 URL 授权判定。
2. 认证成功的信息存在 SecurityContextHolder 的 SecurityContext 里，其内部是 ThreadLocal，随请求线程传递。异步线程是新线程，默认没有这份 ThreadLocal 上下文，所以取不到当前用户，需在提交任务前显式捕获传递，或把所需用户信息作为参数传下去。
3. 前缀标识密文所用的编码器，如 {bcrypt}、{noop}，DelegatingPasswordEncoder 靠前缀分发到对应编码器比对。这样新旧算法的密文可以共存，迁移时登录先按老算法校验、成功后按新格式重写密文逐步洗库，而不必把编码器改回兼容明文。无前缀的老密文对不上委托编码器，就会报 no PasswordEncoder mapped for id "null"。
4. JWT 自包含、服务端不存会话，验签即认证，签发后在有效期内无法直接作废；要拉黑名单就等于把状态又加了回来，失去了无状态的意义。折中是短有效期访问令牌 + 刷新令牌：访问令牌过期快、泄露窗口小，刷新令牌可控、可撤销，配套复杂度是必须接受的代价。
5. 5.7 起官方主推组件式写法：暴露 SecurityFilterChain Bean，不再继承 WebSecurityConfigurerAdapter。继承式配置难以灵活组合多条链（不同 URL 用不同口径），组件式可按 URL 定义多条链、与 authorizeHttpRequests/AuthorizationManager 体系配合，配置更清晰、可组合。

## 10. 关联知识

- [Spring-Boot](开发语言/Java/Spring/Spring-Boot.md)：自动配置与项目基线。
- [REST 接口设计](工程实践/REST接口设计.md)：401/403 与统一错误口径的设计侧约定。

- [国密与加密体系](工程实践/国密与加密体系.md)：国密算法与银行敏感字段落库加密落地。
