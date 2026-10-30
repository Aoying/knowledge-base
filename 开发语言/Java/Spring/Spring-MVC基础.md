---
知识点: Spring MVC基础
层级: 基础
适用版本: Spring Boot 2.7.3
更新日期: 2026-10-31
关联:
  - Spring-Boot
  - Spring-IOC与AOP
  - 接口设计与文档
  - Bean-Validation
  - Java-异常体系
标签:
  - Java
  - 实践
---

# Spring MVC基础

## 1. 核心概念

Spring MVC 是 Spring 的 Web 层框架，职责是「把 HTTP 请求翻译成 Java 方法调用，再把返回值翻译回 HTTP 响应」。Spring Boot 项目里只要引入 `spring-boot-starter-web`，内嵌 Tomcat 与 MVC 基础设施就由自动配置装配好，不用手写 web.xml 或 DispatcherServlet 注册。

一次请求的完整流程（排障时按这个顺序定位）：

1. 请求到达内嵌 Tomcat，被 `DispatcherServlet`（前端控制器）接住。
2. `HandlerMapping` 按 URL + HTTP 方法找到对应的 Controller 方法（Handler）。
3. `HandlerAdapter` 负责真正调用：先解析参数（路径变量、查询参数、请求体、请求头），再执行方法。
4. 方法返回后，`HandlerAdapter` 处理返回值：`@ResponseBody`（或 `@RestController`）走 `HttpMessageConverter` 序列化成 JSON；否则走视图解析。
5. 拦截器（`HandlerInterceptor`）在步骤 3 前后切入；异常则由异常解析链（`@ExceptionHandler` / `@RestControllerAdvice`）接管。

记住这条链，就能判断问题出在哪一环：404 是第 2 步没找到 Handler，参数绑定报错是第 3 步，返回格式不对是第 4 步，统一异常没生效通常是第 5 步的配置问题。

## 2. 基础知识

### Controller 与返回值

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getById(@PathVariable Long id) {
        return userService.getById(id);
    }

    @PostMapping
    public User create(@Valid @RequestBody UserForm form) {
        return userService.create(form);
    }
}
```

- `@Controller` + 方法上的 `@ResponseBody` 等价于类上的 `@RestController`；写接口一律用 `@RestController`，别混用。
- 返回对象默认由 Jackson 序列化成 JSON；返回 `String` 时注意：`@RestController` 下返回的是字符串本身（Content-Type 为 text/plain），不是 JSON 字符串。

### 参数绑定速记

| 注解 | 绑定来源 | 要点 |
|---|---|---|
| `@PathVariable` | URL 路径段，如 `/users/{id}` | 变量名须与占位符一致，或显式写名字 |
| `@RequestParam` | 查询参数 / 表单字段 | 默认 `required = true`，缺参直接 400；可选参数写 `required = false` 或给 `defaultValue` |
| `@RequestBody` | 请求体 JSON | 只能用一次（输入流已消费）；配合 `@Valid` 做校验 |
| `@RequestHeader` | 请求头 | 取 token、traceId 这类元数据 |
| `@ModelAttribute` | 查询参数 + 表单，绑定到对象 | GET 查询条件封装成对象时用；可省略不写 |

### 与 Bean-Validation 配合

```java
public class UserForm {
    @NotBlank(message = "姓名不能为空")
    private String name;

    @Min(value = 0, message = "年龄不能为负")
    private Integer age;
    // getter / setter 省略
}
```

- 入参对象上标 `@Valid`，校验失败抛 `MethodArgumentNotValidException`，由全局异常处理统一转成错误响应，不要在每个方法里接 `BindingResult` 手工判（见第 4 节）。
- 校验规则本身的写法见 [Bean-Validation](开发语言/Java/Bean-Validation.md)。

## 3. 进阶知识

### 拦截器：HandlerInterceptor

拦截器是 MVC 层的横切入口，适合登录校验、日志、traceId 透传这类「请求级」逻辑。

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        // preHandle 返回 false 表示拦截：请求到此为止，不再进 Controller
        if (!hasValidToken(request)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }
        return true;
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final AuthInterceptor authInterceptor;

    public WebConfig(AuthInterceptor authInterceptor) {
        this.authInterceptor = authInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login", "/actuator/**");
    }
}
```

三个方法的分工：`preHandle`（进 Controller 前，返回 false 即拦截）、`postHandle`（Controller 执行后、渲染前）、`afterCompletion`（整个请求完成后，适合记耗时日志、清理 ThreadLocal）。

拦截器、过滤器、AOP 的选型界线：过滤器（Servlet 层）只能看到原始请求响应，适合编码、跨域这类与业务无关的事；拦截器能拿到 Handler（知道是哪个 Controller 方法），适合登录、权限这类「按接口区分」的逻辑；AOP 能切任意方法（含 Service 层），见 [Spring-IOC与AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)。

### 内容协商与 REST

- 客户端用 `Accept` 头声明想要的格式，服务端用 `produces` / `consumes` 声明能处理的类型，不匹配时返回 406 / 415——接口联调时先看这两个头，别一上来就怀疑业务代码。
- 序列化由 `HttpMessageConverter` 负责：JSON 默认 Jackson，`MappingJackson2HttpMessageConverter`；返回对象里不想暴露的字段用 `@JsonIgnore`，日期格式用 `@JsonFormat`。
- REST 风格约定：资源用名词复数（`/api/users`），动作用 HTTP 方法表达（GET 查、POST 增、PUT 全量改、PATCH 部分改、DELETE 删）；接口设计规范见 [接口设计与文档](工程实践/接口设计与文档.md)。

### 文件上传

```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) throws IOException {
    if (file.isEmpty()) {
        throw new BizException("文件为空");
    }
    String name = file.getOriginalFilename();
    if (name == null || name.isBlank()) {
        throw new BizException("文件名非法");
    }
    Path target = Paths.get("/data/upload", name);
    file.transferTo(target);
    return target.toString();
}
```

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 20MB
```

- 超过限制抛 `MaxUploadSizeExceededException`，同样进全局异常处理，别让它裸奔成 500。
- 上传路径、文件名校验（防路径穿越）是生产必做项，示例里的 `/data/upload` 只是示意。

## 4. 实战应用：全局异常处理

```java
@Slf4j  // 需 Lombok；不用 Lombok 时自行定义 Logger
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));
        return Result.fail(msg);
    }

    @ExceptionHandler(BizException.class)
    public Result<?> handleBiz(BizException e) {
        return Result.fail(e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result<?> handleUnknown(Exception e) {
        log.error("未知异常", e);
        return Result.fail("系统繁忙，请稍后重试");
    }
}
```

- `Result` 为项目统一返回体示意；`BizException` 为业务异常示意，异常体系的完整分层见 [Java-异常体系](开发语言/Java/Java-异常体系.md)。
- 兜底的 `Exception.class` 处理器必须有，否则未知异常会以 Tomcat 默认错误页 / 白页形式出去。
- 异常处理器里不要再抛异常，也不要在里面调可能失败的外部服务——异常处理本身出问题最难排查。

## 5. 疑难杂症

- **404 但 Controller 明明写了**：先查三点——启动日志里有没有映射上（搜 `Mapped` 日志）、HTTP 方法对不对（GET 调了 POST 接口会 405 不是 404）、类上有没有漏 `@RestController` 或包扫描扫不到（启动类所在包之外）。
- **400 参数绑定失败**：`@RequestParam` 缺参、类型对不上（传了 abc 给 Long）、`@RequestBody` 的 JSON 字段类型错——看异常信息里的字段名定位，别猜。
- **拦截器里读了 body，后续读不到**：请求输入流只能读一次，拦截器里要读 body 须包一层可重复读的 RequestWrapper；只做登录校验的拦截器别碰 body。
- **静态资源 / actuator 被拦截器挡住**：`excludePathPatterns` 里把 `/static/**`、`/actuator/**`、登录接口都排除，见第 3 节示例。
- **返回中文乱码**：确认 `produces` 与响应编码，Spring Boot 默认 UTF-8，乱码多半是反向代理或前端处理问题，先看原始响应头。

## 6. 常见误区

- **`@Controller` 当 `@RestController` 用**：返回对象时被当成视图名解析，报「找不到视图」而不是返回 JSON。
- **`@RequestParam` 以为可选**：默认必填，缺参直接 400；可选参数必须显式 `required = false`。
- **在拦截器里做重业务**：拦截器是请求必经之路，只做轻量校验；调数据库、调下游接口放 Service 层。
- **异常处理只覆盖业务异常**：校验异常、类型转换异常、上传超限这些框架异常没接住，照样以 500 裸奔。
- **跨域靠 `@CrossOrigin` 到处贴**：单个接口调试可以，生产统一在 `WebMvcConfigurer.addCorsMappings` 里配，别散落在各 Controller 上。

## 7. 版本差异

- Boot 2.7 线用 Spring Framework 5.3，Servlet 规范为 `javax.*`；Boot 3 线起换成 `jakarta.*`，升级时 `javax.servlet` 导入要全量改包名，见 [Spring Boot](开发语言/Java/Spring/Spring-Boot.md) 的升级路线图。

## 小技巧与冷知识

【小技巧】可选参数别用基本类型：`@RequestParam(required = false) int id` 缺参时会直接报错，因为 Spring 没法给 int 赋 null；改成 `Integer` 才能在缺参时拿到 null。反过来也一样——必填参数用 `int` 没问题，缺参本来就该 400；只有写了 `required = false` 或 `defaultValue` 的参数才必须用包装类型。

【小技巧】同一个 URL 还能按查询参数或请求头再分流：`@GetMapping(value = "/users", params = "version=1")` 和 `params = "version=2"` 可以是两个方法，`headers = "X-Client=app"` 同理。老接口想按版本号或客户端分流时，不用在方法里写 if 判断版本，用映射条件本身分流更干净；条件冲突时 Spring 会报映射歧义，启动就暴露。

【小技巧】每个请求都要的公共数据用方法级 `@ModelAttribute`：在 Controller 里写一个 `@ModelAttribute void fill(Model model)`，每次进该 Controller 的方法前它都会先跑，把当前用户、公共下拉项放进 Model，视图和后续方法都能直接用。比在每个方法开头重复查一次省事，也比拦截器里 setAttribute 再强转类型安全。

【小技巧】想让某个业务异常直接带状态码，不用写异常处理器：在异常类上标 `@ResponseStatus(HttpStatus.NOT_FOUND)`，抛它就直接返回 404，`reason` 还能顺带定错误文案。只对“状态码固定、文案固定”的异常用这招；需要统一错误体格式、要记日志的还是走 `@RestControllerAdvice`，两者别混着用。

【小技巧】新建资源返回 201 加 Location 头用 `ResponseEntity`：`ResponseEntity.created(URI.create("/api/users/" + id)).body(user)`，状态码、Location 头、响应体一次给齐。直接返回对象默认是 200，创建接口返回 200 虽然能用，但按 REST 约定该给 201；前端按状态码判断“是不是新建成功”时，这一行就是依据。

【小技巧】日期字符串绑不上先看 `@InitBinder`：入参里的 `LocalDateTime` 传 `2026-10-31 10:00:00` 这种格式默认解析失败，在 Controller 里写 `@InitBinder` 注册一个 `DateTimeFormatter`，或在字段上标 `@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")`。400 报错先看是不是日期格式问题，比怀疑前端传错字段快得多；全局统一格式用 `WebMvcConfigurer` 里配 `addFormatters`，别每个 Controller 各写一套。

【小技巧】想让 Controller 方法直接拿到“当前用户”这类对象，用自定义参数解析器：实现 `HandlerMethodArgumentResolver`，在 `supportsParameter` 里认出标了 `@CurrentUser` 的参数，`resolveArgument` 里从请求头或 ThreadLocal 取值。比拦截器里 `request.setAttribute` 再在每个方法里强转干净，类型不对编译期就能发现，参数解析失败也走统一的 400 链路。

【小技巧】`@SessionAttribute` 和 `@SessionAttributes` 是两回事，别混用：前者是读 session 里已经存在的值（`@SessionAttribute("user") User user`），没有就报异常；后者是把 Model 里的属性暂存进 session、跨多次请求用，常见于多步表单向导。只想读 session 用前者，想“这次放、几次后取”才用后者。

【小技巧】上传大文件用 `transferTo` 落盘，别用 `getBytes()`：`transferTo` 是把上传内容直接写到目标文件，内存里不攒；`getBytes()` 会把整个文件读进一个 byte 数组再处理，几百 MB 的文件直接把堆顶满。原文第 3 节的上传示例用的是 `transferTo`，判断“上传接口 OOM”时先看代码里是不是有人顺手调了 `getBytes()`。

【小技巧】可选的请求头用 `required = false` 或给默认值：`@RequestHeader(value = "X-Trace-Id", required = false) String traceId`，没带这个头的请求也能进方法，traceId 是 null；不写 required 默认是 true，缺头直接 400。网关透传的 traceId、客户端版本号这类“有就用、没有也行”的头，都要显式标成可选。

【小技巧】同一个异常有多个 `@ExceptionHandler` 时，最具体的那个先生效：`@ExceptionHandler(BizException.class)` 和 `@ExceptionHandler(Exception.class)` 同时存在，抛 BizException 会走前者而不是兜底的后者。写全局异常处理时先把具体异常的处理方法写全，再留一个 `Exception.class` 兜底；顺序写反了也不怕，Spring 是按异常类型匹配度选的，不是按方法声明顺序。

【冷知识】`@RestController` 是 Spring 4.0 才引入的：2013 年 12 月发布的 Spring 4.0 之前，写接口必须 `@Controller` 加每个方法上的 `@ResponseBody`，新注解只是把两者合二为一。看 2013 年之前的老项目或老博客时满眼都是 `@ResponseBody`，不是写法落后，是当时还没有这个组合注解。

【冷知识】URL 里的分号默认会被截掉，矩阵变量开箱不可用：`/users/123;version=2` 这种写法里，分号后面的内容 Spring 默认直接丢掉（`UrlPathHelper.removeSemicolonContent` 默认 true），`@MatrixVariable` 拿不到值。想用矩阵变量要在 `WebMvcConfigurer` 里把 `configurePathMatch` 的 `setRemoveSemicolonContent(false)` 打开；不打开也别硬拼分号参数，Spring 根本不会把它当成请求的一部分。

## 8. 面试与自测

1. 一个请求从到达 Tomcat 到返回 JSON，DispatcherServlet 链路上依次经过哪些组件？404 和参数绑定失败分别卡在哪一步？
2. 拦截器、过滤器、AOP 都能做横切逻辑，登录校验应该选哪个？为什么？
3. `@Valid` 校验失败后异常去哪了？为什么推荐用 `@RestControllerAdvice` 统一处理，而不是每个方法里接 BindingResult？

### 参考答案

1. 依次是：Tomcat → DispatcherServlet → HandlerMapping（找 Handler）→ HandlerAdapter（解析参数、调方法）→ HttpMessageConverter（序列化返回值）。404 卡在 HandlerMapping（没找到匹配的 URL + 方法）；参数绑定失败卡在 HandlerAdapter 解析参数时。
2. 选拦截器。过滤器在 Servlet 层，只能看到原始请求响应，不知道是哪个接口；AOP 能切任意方法但偏重，且拿不到 HTTP 语义；拦截器能拿到 Handler（知道是哪个 Controller 方法），适合「按接口区分」的登录、权限逻辑。
3. 校验失败抛 `MethodArgumentNotValidException`，由 `@RestControllerAdvice` 里的 `@ExceptionHandler` 接住并转成统一错误响应。每个方法接 BindingResult 会把校验处理散落在各处，漏接一个就裸奔成 500；统一处理保证所有接口的错误格式一致。

## 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
- [Spring-IOC与AOP](开发语言/Java/Spring/Spring-IOC与AOP.md)
- [接口设计与文档](工程实践/接口设计与文档.md)
- [Bean-Validation](开发语言/Java/Bean-Validation.md)
- [Java-异常体系](开发语言/Java/Java-异常体系.md)
