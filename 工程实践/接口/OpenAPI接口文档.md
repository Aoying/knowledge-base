---
知识点: OpenAPI 接口文档
层级: 基础
适用版本: springdoc-openapi 1.7.x / Spring Boot 2.7.3
更新日期: 2026-10-23
关联:
  - Spring-Boot
  - Bean-Validation
标签:
  - Java
  - 实践
  - 速查
---

# OpenAPI 接口文档

## 1. 核心概念

手写接口文档必过期：字段改了文档没跟，前后端联调靠口头对齐。OpenAPI 方案是从代码（路由 + 注解 + DTO）自动生成标准描述（JSON/YAML），再渲染成交互页面（Swagger UI），文档与代码同源、随版本发布。

Boot 2.7 的选型结论：用 **springdoc-openapi**，不要再上 springfox——springfox 已多年不维护、与新版本 Spring 的路径匹配策略冲突是经典启动报错来源。springdoc 1.7.x 是对应 Boot 2.7 的版本线（Boot 3 要换 springdoc 2.x + Jakarta，别混装）。

## 2. 基础知识

引入依赖（版本由 1.7.x 管理，勿在 Boot 2.7 上用 2.x）：

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>
```

启动后默认地址：文档 JSON `/v3/api-docs`，页面 `/swagger-ui.html`（自动跳 `/swagger-ui/index.html`）。基础信息配置：

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI appOpenAPI() {
        return new OpenAPI().info(new Info()
                .title("神农 e 贷法人版 API")
                .version("v1")
                .description("客户、额度、进件、审批接口"));
    }
}
```

常用注解（都在 `io.swagger.v3.oas.annotations` 包）：

| 注解 | 位置 | 作用 |
|---|---|---|
| `@Tag(name, description)` | Controller 类 | 接口分组名与说明 |
| `@Operation(summary, description)` | 方法 | 接口标题与详细说明 |
| `@Parameter(description, required)` | 方法参数 | 路径/查询参数说明 |
| `@Schema(description, example)` | DTO 字段 | 字段含义与示例值 |
| `@ApiResponse` | 方法 | 补充非默认响应码说明 |

```java
@Tag(name = "进件管理", description = "贷款申请提交与查询")
@RestController
@RequestMapping("/api/applies")
public class ApplyController {

    @Operation(summary = "提交进件", description = "校验通过后落库并启动审批流程")
    @PostMapping
    public Result<Long> submit(@RequestBody @Valid ApplySubmitDTO dto) { ... }

    @Operation(summary = "按编号查询进件")
    @GetMapping("/{id}")
    public Result<ApplyVO> get(
            @Parameter(description = "进件编号", required = true) @PathVariable Long id) { ... }
}
```

## 3. 进阶知识

### 与 Bean-Validation 的配合

springdoc 会自动把 DTO 上的校验注解翻译成 Schema 约束，前后端对字段规则有一份共同事实，不用在文档里再手写一遍：

| 校验注解 | 生成的约束 |
|---|---|
| `@NotNull` / `@NotBlank` / `@NotEmpty` | `required: true` |
| `@Size(max = 32)` / `@Length` | `maxLength` / 数组 `maxItems` |
| `@Min` / `@Max` / `@DecimalMin` 等 | `minimum` / `maximum` |
| `@Pattern` | `pattern` 正则 |
| `@Email` | `format: email` |

口径注意：校验注解是**运行时事实**（见 [Bean-Validation](开发语言/Java/Bean-Validation.md)），文档只是它的投影；只在 `@Schema` 里写「必填」而不加校验注解，文档好看但接口照样放行非法数据，别把顺序弄反。

### 文档分组

多组接口按业务域拆组，页面顶部可切换，前端按组对接：

```java
@Bean
public GroupedOpenApi applyGroup() {
    return GroupedOpenApi.builder()
            .group("进件与审批")
            .pathsToMatch("/api/applies/**", "/api/approvals/**")
            .build();
}

@Bean
public GroupedOpenApi customerGroup() {
    return GroupedOpenApi.builder()
            .group("客户与额度")
            .packagesToScan("com.example.customer.controller")
            .build();
}
```

分组维度二选一（路径或包），团队内统一一种，别混用导致接口重复出现或漏组。

### 全局鉴权声明

接口要带 Token 时在配置里声明安全方案，页面上出现 Authorize 按钮，联调时填一次全局生效：

```java
new OpenAPI().components(new Components().addSecuritySchemes("bearerAuth",
        new SecurityScheme().type(SecurityScheme.Type.HTTP).scheme("bearer").bearerFormat("JWT")))
    .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
```

网关统一鉴权的系统也要照此声明：文档描述的是调用契约，不是鉴权实现位置。

## 4. 实战应用

- **环境开关**：生产与对外环境关掉页面或加访问控制——`springdoc.swagger-ui.enabled` 与 `springdoc.api-docs.enabled` 按 profile 配置；内网生产至少限制到运维网段，接口文档等于把系统地图公开，别裸奔。
- **DTO 别偷懒**：入参出参用独立 DTO/VO 并写 `@Schema` 说明与示例；直接拿 Entity 当出参，字段语义不清还会把内部字段（逻辑删除标志、审计字段）暴露给前端。
- **枚举与字典**：枚举字段在 `@Schema` 里写清每个取值的业务含义（如审批状态 0 待审 / 1 通过 / 2 驳回），这是联调吵架最多的地方。
- **导出给前端**：`/v3/api-docs` 的 JSON 可导入 Apifox/Postman 生成用例与 Mock；接口变更评审时 diff 这份 JSON，比口头通知可靠。

## 5. 疑难杂症

- **页面 404 或打开空白**：先直连 `/v3/api-docs` 看 JSON 是否正常——JSON 正常而页面打不开多是静态资源被网关/拦截器拦了（放行 `/swagger-ui/**`、`/v3/api-docs/**`）；JSON 都没有则查依赖是否引入成功、是否被 profile 开关关掉。
- **接口没出现在文档里**：Controller 没被组件扫描到、方法不是公开映射、或分组的路径/包规则没覆盖它。
- **泛型与嵌套对象展开不对**：返回包装类（如统一 `Result<T>`）要保证泛型实参明确；实在展开不了的复杂结构用 `@Schema(implementation = ...)` 指定呈现模型，别为了文档改业务结构。
- **文件上传接口**：参数用 `MultipartFile` 并声明 `consumes = MediaType.MULTIPART_FORM_DATA_VALUE`，否则 UI 上没有文件选择框。

## 6. 常见误区

- 注解写满整个 Controller，把业务代码淹没：说明文字保持短句，复杂业务规则写在项目 Wiki/需求文档里并在 `description` 放链接。
- 为了「文档好看」在 `@Schema` 里写与校验注解不一致的约束（文档说必填、实际不校验），联调必翻车。
- 生产环境开着文档还无鉴权，且示例值里带真实客户数据——示例一律用虚构值。

## 7. 面试与自测

1. springdoc 与 springfox 的现状区别？Boot 2.7 该选哪个版本线？
2. 校验注解是怎么变成文档约束的？为什么不能只写 `@Schema` 不写校验？
3. 文档分组有哪两种维度？混用会出什么问题？
4. 生产环境接口文档应该怎么处理？风险是什么？

### 参考答案

1. springfox 已多年不维护，且与新版本 Spring 的路径匹配策略冲突，是经典启动报错来源；springdoc 是当前主流。Boot 2.7 该选 springdoc 1.7.x 版本线，Boot 3 要换 springdoc 2.x 加 Jakarta，两条线别混装。
2. springdoc 会自动把 DTO 上的校验注解翻译成 Schema 约束，如 @NotNull/@NotBlank 对应 required、@Size 对应 maxLength、@Min/@Max 对应 minimum/maximum、@Pattern 对应 pattern。不能只写 @Schema 不写校验：校验注解是运行时事实，文档只是它的投影；只在 @Schema 里写必填而不加校验，文档好看但接口照样放行非法数据，顺序不能反。
3. 两种维度：按路径（pathsToMatch）或按包（packagesToScan）。团队内必须统一一种，二选一；混用会导致接口重复出现或漏组。
4. 生产与对外环境应关掉页面或加访问控制，用 springdoc.swagger-ui.enabled 与 springdoc.api-docs.enabled 按 profile 配置，内网生产至少限制到运维网段。风险是接口文档等于把系统地图公开，裸奔等于把接口、字段、枚举含义全部暴露；示例值一律用虚构值，不能带真实客户数据。
## 8. 关联知识

- 前置：[Spring Boot](开发语言/Java/Spring/Spring-Boot.md)、[Bean-Validation](开发语言/Java/Bean-Validation.md)
- 相关：[单元测试与Mock](工程实践/单元测试与Mock.md)（`@WebMvcTest` 验证校验与文档契约）
- 延伸：OpenAPI 契约优先（先写 YAML 再生成代码）、接口版本治理
