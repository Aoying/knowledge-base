---
知识点: Bean Validation 参数校验
层级: 基础
适用版本: Spring Boot 2.7
更新日期: 2026-10-22
关联:
  - Java
  - Spring-Boot
标签:
  - Java
  - 实践
---

# Bean Validation 参数校验

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

