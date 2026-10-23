---
知识点: Bean Validation 参数校验
层级: 基础
适用版本: Spring Boot 2.7
更新日期: 2026-10-23
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

## 关联知识

- [Spring Boot](开发语言/Java/Spring-Boot.md)
