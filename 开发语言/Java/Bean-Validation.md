---
知识点: Bean Validation 参数校验
层级: 基础
适用版本: Spring Boot 2.7
更新日期: 2026-10-31
关联:
  - Java
  - Spring-Boot
标签:
  - Java
  - 实践
---

# Bean Validation 参数校验

## 1. 核心概念

Bean Validation（JSR 380，校验注解规范）+ Hibernate Validator（默认实现）的分工是：注解声明「字段长什么样才合法」，框架在入参绑定后自动校验，业务代码不再写满屏 if 判空。它只管**格式与范围这类无状态规则**（非空、长度、正则、数值区间）；「客户状态必须正常」「额度够不够」这类要查库的业务规则不在它的范围，仍放 Service。

```xml
<!-- Boot 2.3+ 必须显式加，web starter 里没有了 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## 2. 常用注解速记

| 注解 | 作用 | 注意 |
|---|---|---|
| `@NotNull` | 不能为 null | 对字符串拦不住空串 |
| `@NotBlank` | 字符串不能为 null/纯空白 | 只能用在 CharSequence 上 |
| `@NotEmpty` | 集合/字符串长度 > 0 | 拦不住纯空白字符串 |
| `@Size(min,max)` | 字符串长度/集合大小 | 用在数值上会校验失败 |
| `@Pattern(regexp)` | 正则 | 写错正则启动即报错，先本地验 |
| `@DecimalMin/@DecimalMax` | BigDecimal 区间，字符串参数 | 比用 double 的 `@Min` 安全 |
| `@Min/@Max` | 整数区间 | 用 long 传参 |
| `@Email` | 邮箱格式 | 只验格式，不验域名真实性 |
| `@Past/@Future` | 日期在过去/未来 | 用在生日、到期日 |
| `@Positive/@Negative` | 正数/负数 | JDK 与版本差异小，可放心用 |

区分 `@NotNull/@NotEmpty/@NotBlank` 是最常见的面试与实战混淆点：校验字符串必填一律 `@NotBlank`，只用 NotNull 会漏掉 `""`。

## 3. 触发与级联

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
```

- `@Valid` 只在 Controller 入参 / `@Validated` 的 Service 方法上才触发；DTO 字段上的注解本身不会"自动"生效。
- 嵌套对象要单独标 `@Valid` 才会级联进去校验；嵌套集合校验要双写：字段上 `@Valid` + 泛型上注解（`List<@NotBlank String>`），否则集合元素不校验。
- 非 `@RequestBody` 的散参数（`@RequestParam`、路径变量、方法参数直接是基本类型）要校验，注解标在参数上，且类上必须加 `@Validated`，触发的异常也换成 `ConstraintViolationException`，全局处理要单独接。
- Service 层方法校验同样靠类上 `@Validated` + 参数注解，适合内部接口的二次防线。

## 4. 统一异常返回

```java
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

    @ExceptionHandler(ConstraintViolationException.class)
    public R<?> handle(ConstraintViolationException e) {
        return R.fail("参数校验失败: " + e.getMessage());
    }
}
```

两个异常都要接：`@RequestBody` 校验失败抛 `MethodArgumentNotValidException`（表单绑定是 `BindException`），散参数/方法校验失败抛 `ConstraintViolationException`。只接一个，另一条路径就会漏成 500，是联调时最常见的「校验没生效」假象——其实校验生效了，只是异常没人接。

## 5. 分组与自定义校验

- 新增/修改共用 DTO 时用分组：`@Validated(Create.class)` / `@Validated(Update.class)`，`@NotNull(groups = Update.class)` 按场景生效。分组接口是普通空接口；注意一旦用了分组，未指定 groups 的注解属于默认组 `Default`，触发时没带 Default 就不会校验，这是分组最容易踩的坑——要么全字段都标组，要么触发时把 `Default.class` 一起带上。
- 跨字段规则（如「结束日期必须晚于开始日期」）单个字段注解表达不了，用类级自定义注解：写一个标在类上的注解 + `ConstraintValidator` 实现，在 `isValid` 里拿整个对象判断。
- 与数据库相关的唯一性等校验（如证件号不能重复）不要塞进 Validator 里查库：校验层会变得难测且时机不对（经验：放 Service，且数据库唯一索引兜底）。

## 6. 快速失败

默认会把所有字段错误收集完再抛。入参层层嵌套、校验很重时可开快速失败：给 `Validator` 配置 `fail_fast=true`（Hibernate Validator 属性），第一个错误就返回。网关层/对外接口更适合快速失败，省时间也少暴露内部结构。

## 7. 边界

- 校验是第一道防线，不是唯一防线：金额、状态机流转这类业务规则放 Service 里再查一遍。
- message 里别拼用户输入，防日志注入；对外返回的错误信息只给字段名与规则描述，不回显完整入参。


## 小技巧与冷知识

【小技巧】`@Valid` 不支持分组，`@Validated` 才支持：标准的 `@Valid` 没有 groups 属性，想按新增/修改分组校验，必须用 Spring 的 `@Validated(Create.class)`。Controller 入参只写 `@Valid` 时，分组注解永远按默认组跑，分组写了等于白写；原文第 5 节的分组用法，触发端必须配 `@Validated` 才能生效。

【小技巧】`@Pattern` 的正则不用写 `^...$`：Bean Validation 默认用全串匹配（`Matcher.matches()`），正则本来就要整个字符串都合上才算通过，两端锚点是多余的。原文手机号例子里的 `^1[3-9]\d{9}$` 去掉 `^` 和 `$` 效果完全一样；少了锚点，正则也更短、更好读。

【小技巧】`BindingResult` 紧跟 `@Valid` 参数时不抛异常：想在方法里自己处理校验错误（比如只返回第一个字段的错误），在被校验参数后面立刻加一个 `BindingResult` 参数，校验失败就不再抛 `MethodArgumentNotValidException`，而是把错误装进它里面等你取。位置必须紧跟被校验参数，中间隔一个别的参数就失效，又会回到抛异常的老路。

【小技巧】`@NotBlank` 标在数字上会直接报错：它只能用在 CharSequence 上，标在 `Integer`、`Long` 这类字段上，校验跑到它时会抛 `UnexpectedTypeException`，而不是“拦不住”。数字必填用 `@NotNull`，范围用 `@Min`/`@Max`；把字符串的注解顺手复制到数字字段上，是复制粘贴时最常见的翻车。

【小技巧】`message` 里能插注解的属性值：`@Size(min = 2, max = 20, message = "长度必须在 {min} 到 {max} 之间")`，`{min}`、`{max}`、`{regexp}` 这些占位符会被替换成注解上写的实际值。改了注解的数值不用再改文案，提示语和规则永远对得上；原文例子里的 message 全是写死数字，规则一改文案就跟着过期。

【小技巧】快速失败的属性全名是 `hibernate.validator.fail_fast`：原文第 6 节只写了 `fail_fast`，创建 `ValidatorFactory` 时要传这个全名才生效，配置文件里只写 `fail_fast=true` 是配不上的。Spring Boot 里没有现成的开关，要自定义 `LocalValidatorFactoryBean` 或在建 Validator 时显式传这个属性；配完拿一个非法 DTO 测一下，只报一个错才是真的开了快速失败。

【小技巧】单元测试里直接校验 DTO，不用启动 Spring：`Validation.buildDefaultValidatorFactory().getValidator().validate(dto)` 返回 `Set<ConstraintViolation>`，几行代码就能验证注解写没写对、分组生没生效。比起调一次接口再看返回 400，这招在测试里定位“到底是哪个注解没生效”快得多；DTO 改了注解，先跑这个测试再联调。

【冷知识】`@Size` 数的是 UTF-16 代码单元，不是用户看到的字符数：一个 emoji 在 Java 字符串里占两个 char，`@Size(max = 10)` 的字段里放 6 个 emoji 就超了。按“用户看到的字符数”限长得自己写校验，用 `codePointCount` 数码点；昵称、备注这类允许 emoji 的字段直接用 `@Size` 限长，边界上会比预期严一倍。

【冷知识】Bean Validation 一共三代：JSR 303（1.0，2009 年）、JSR 349（1.1，2013 年）、JSR 380（2.0，2017 年，新增容器元素约束和 `@NotBlank`、`@PositiveOrZero` 这些注解）。3.0 改名 Jakarta Validation，包名从 `javax.validation` 改成 `jakarta.validation`；Spring Boot 2.7 用的还是 2.0 的 `javax` 包，照着新版文档抄导入会编译不过。

【小技巧】`@Email` 默认只验格式骨架，想卡具体域名要自己收紧：它不管域名是不是真实存在，也不限制后缀，用 `@Email(regexp = ".*@company\\.com$")` 才能限定只收公司邮箱。原文第 2 节说了“只验格式，不验域名真实性”，落地时要么接受这个宽松度、要么用 `regexp` 属性或再叠一个 `@Pattern` 收紧，别指望默认的 `@Email` 替你挡住随便填的邮箱。

## 8. 面试与自测

1. 字符串必填该用 `@NotNull`、`@NotEmpty` 还是 `@NotBlank`？三者各拦不住什么？
2. 嵌套对象和集合元素校验经常漏写什么，导致“校验没生效”？
3. `@RequestBody` 入参和散参数（`@RequestParam`、路径变量）校验失败抛的异常一样吗？全局处理要接哪几个？

### 参考答案

1. 字符串必填一律 `@NotBlank`：`@NotNull` 只拦 null，拦不住 `""`；`@NotEmpty` 拦不住纯空白字符串；`@NotBlank` 拦 null 与纯空白，但只能用在 CharSequence 上。
2. 嵌套对象漏标 `@Valid` 就不会级联校验；集合元素要双写——字段上 `@Valid` 加泛型上注解（如 `List<@NotBlank String>`），只写一个集合元素不校验。
3. 不一样：`@RequestBody` 失败抛 `MethodArgumentNotValidException`（表单绑定是 `BindException`），散参数/方法校验失败抛 `ConstraintViolationException`，后者还要求类上加 `@Validated`。两个都要在全局处理里接，只接一个另一条路径会漏成 500。

## 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
