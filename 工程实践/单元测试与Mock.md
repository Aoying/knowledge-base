---
知识点: 单元测试与 Mock
层级: 基础
适用版本: JUnit 5 / Mockito 3+ / Spring Boot 2.7.3
更新日期: 2026-10-23
关联:
  - Spring-Boot
  - CICD
标签:
  - Java
  - 实践
---

# 单元测试与 Mock

## 1. 核心概念

测试的价值不是「跑绿」，而是给重构和改需求兜底：有一批可信的测试，改审批规则、动额度计算时敢下手。测试分三层，成本和收益完全不同：

| 层 | 对象 | 数量占比 | 特点 |
|---|---|---|---|
| 单元测试 | 一个类/方法 | 最多（约七成） | 毫秒级、无外部依赖、定位精确 |
| 切片/集成测试 | 一组组件 + 真实框架装配 | 适量 | 用测试容器或嵌入式库验证配置、SQL、事务 |
| 端到端测试 | 完整链路 | 极少 | 慢、脆，只保核心主流程 |

倒金字塔（全靠手工点页面和 E2E）是多数老项目的现状：慢、没人跑、坏了也不敢信。本篇只讲前两层在 Spring Boot 项目的落地。

## 2. 基础知识

`spring-boot-starter-test` 已打包 JUnit 5、Mockito、AssertJ、MockMvc。JUnit 5 基本件：

```java
class QuotaCalculatorTest {

    @BeforeEach
    void setUp() { /* 每个用例前重建对象，保证用例互相独立 */ }

    @Test
    @DisplayName("可用额度不足时应拒绝")
    void shouldRejectWhenQuotaNotEnough() {
        QuotaCalculator calc = new QuotaCalculator();
        assertThrows(BizException.class,
                () -> calc.check(new BigDecimal("100"), new BigDecimal("50")));
    }

    @ParameterizedTest
    @CsvSource({ "100, 100, true", "100, 101, false" }) // 参数化：一组边界值一网打尽
    void checkBoundary(BigDecimal quota, BigDecimal amount, boolean ok) {
        assertEquals(ok, new QuotaCalculator().canUse(quota, amount));
    }
}
```

命名规范：方法名写清「场景 + 期望」，失败时光看测试名就知道坏了哪条规则。用例之间禁止共享可变状态、禁止依赖执行顺序。

## 3. 进阶知识

### Mockito：只 mock 别人的依赖，不 mock 被测对象

```java
@ExtendWith(MockitoExtension.class)
class ApplyServiceTest {

    @Mock ApplyMapper applyMapper;          // 依赖打桩
    @Mock QuotaService quotaService;
    @InjectMocks ApplyService applyService; // 被测对象，依赖自动注入

    @Test
    void submitShouldDeductQuota() {
        ApplyDTO dto = buildDto();
        when(quotaService.deduct(anyLong(), any()))
                .thenReturn(true);          // 打桩：规定依赖的行为

        applyService.submit(dto);

        verify(applyMapper).insert(any());  // 验证：关键协作确实发生
        ArgumentCaptor<BigDecimal> captor = ArgumentCaptor.forClass(BigDecimal.class);
        verify(quotaService).deduct(eq(7L), captor.capture());
        assertEquals(new BigDecimal("100"), captor.getValue()); // 捕获实参断言金额
    }
}
```

要点：

- `when(...).thenReturn/thenThrow` 定义打桩；`verify` 验证交互；参数捕获适合断言「传给下游的值对不对」。
- 静态方法、final 类 mock 需要 `mockito-inline`（Mockito 3 已内置该能力线，Boot 2.7 的依赖版本按 starter 管理即可）；能不用就不用，靠静态工具类传递状态的设计本身就值得改。
- 滥用 mock 的信号：一个用例写了十几行打桩、改实现细节（没改行为）测试全红。mock 的是**边界**（数据库、远程、时间、随机），不是每个协作者。

### Spring 切片测试：只装需要的层

- `@WebMvcTest(ApplyController.class)`：只装 MVC 层，Service 用 `@MockBean` 顶替，专测参数校验、序列化、状态码，配合 MockMvc。
- 数据层：项目用 MyBatis-Plus，没有 JPA 的 `@DataJpaTest` 等价物；测 Mapper SQL 用 `@SpringBootTest` + 测试库（见下），或 Testcontainers 起真实数据库容器。
- `@SpringBootTest`：装全容器，最接近真实但最慢，只留给少量关键链路（提交一笔申请全流程），不要每个类都用。

数据库测试的口径：神农用 OceanBase（MySQL 模式），H2 内存库语法与方言差异会制造「测试过、线上挂」的假绿；关键 SQL 的测试优先 Testcontainers 或独立测试库，与 [CI/CD](工程实践/运维/CICD.md) 流水线一起跑。

### 与校验、事务的配合

- 参数校验规则的测试放 `@WebMvcTest`：故意传非法参数，断言 400 与错误结构（校验注解见 [Bean-Validation](开发语言/Java/Bean-Validation.md)）。
- 事务回滚测试：测试方法加 `@Transactional` 时框架默认在用例后回滚，方便测写入；但这也意味着「提交后可见性」类问题测不出来，此类用例要显式去掉注解并自己清数据。

## 4. 实战应用

### 覆盖率的取舍

- 用 JaCoCo 出报告进 CI，但**不设全库一刀切的红线**：DTO、配置类、生成代码拉低分母没有意义。
- 分层定目标：金额计算、状态机、规则判断等核心逻辑行覆盖要高且分支要覆盖（if 的两边都走到）；边缘工具类有冒烟用例即可。
- 覆盖率是「哪里完全没测」的探照灯，不是质量证明：断言写得烂的 100% 覆盖一文不值。评审测试代码先看断言有没有对准业务规则。

### 先测什么（存量项目补测试的顺序）

1. 出过线上事故的逻辑——回归用例锁死，别让同一个坑踩两次。
2. 纯计算/规则类：额度、利率、状态流转——无依赖、收益最高。
3. 核心写入链路的集成用例：申请提交、审批通过/驳回。
4. 其他新功能实行「新代码必带测试」，存量随改动逐步补，不停工搞运动式补测。

## 5. 疑难杂症

- **测试互相污染**：静态状态、真实数据库残留、时间没冻结。用例独立三原则：自建数据、自清理、时间/随机用可注入的 Clock/参数而不是静态 now。
- **本地过、CI 挂**：多半是执行顺序依赖或时区/Locale 差异；JUnit 支持随机顺序跑一次复现，在 CI 同款镜像里定位（与 CICD 篇的 flaky 治理呼应）。
- **mock 了半天发现测的是 mock 本身**：断言只验证了自己 when 出来的返回值。修正方向：断言被测对象的输出与状态变化，mock 只负责隔离不稳定边界。

## 6. 常见误区

- 追求覆盖率数字，给每个 getter/setter 写用例，核心规则反而没测。
- 集成测试连真实外部系统（短信、第三方接口）：慢且不稳定，外部边界一律打桩或录制回放。
- 一个测试方法塞五个场景：失败时分不清哪条规则坏了，拆成小用例并用参数化收边界值。

## 7. 面试与自测

1. 单元、切片、端到端三层测试各测什么？为什么要呈金字塔？
2. `@Mock` 与 `@MockBean` 的区别？分别在什么场景出现？
3. 为什么不建议用 H2 测 OceanBase/MySQL 的关键 SQL？
4. 覆盖率 90% 为什么不等于质量高？你怎么定一个项目的覆盖目标？
5. 一个测试时好时坏（flaky），你的排查顺序是什么？

### 参考答案

1. 单元测试测一个类或方法，数量最多（约七成），毫秒级、无外部依赖、定位精确；切片/集成测试测一组组件加真实框架装配，数量适量，验证配置、SQL、事务；端到端测试测完整链路，数量极少，只保核心主流程。要呈金字塔，因为越往上越慢越脆：倒金字塔（全靠手工点页面和 E2E）是多数老项目的现状，慢、没人跑、坏了也不敢信。
2. @Mock 是纯 Mockito 的打桩，不启动 Spring 容器，用于纯单元测试；@MockBean 是 Spring Test 的 Bean 顶替，用于切片测试（如 @WebMvcTest）中替换容器里的 Service，专测参数校验、序列化、状态码。场景分界：不需要 Spring 装配时用 @Mock，需要容器装配、只想替换某一层时用 @MockBean。
3. 因为 H2 内存库与 OceanBase/MySQL 在语法与方言上有差异，会制造测试过、线上挂的假绿。关键 SQL 的测试应优先用 Testcontainers 起真实数据库容器，或独立测试库，并随 CI 流水线一起跑，而不是用 H2 自证通过。
4. 覆盖率是哪里完全没测的探照灯，不是质量证明：断言写得烂的 100% 覆盖一文不值，DTO、配置类、生成代码还会拉低分母。所以不设全库一刀切的红线，改为分层定目标：金额计算、状态机、规则判断等核心逻辑行覆盖要高且分支要覆盖（if 的两边都走到），边缘工具类有冒烟用例即可；评审测试代码先看断言有没有对准业务规则。
5. 排查顺序：先查用例独立性（共享可变状态、真实数据库残留、时间与随机数没冻结，应用三原则是自建数据、自清理、时间与随机用可注入的 Clock 与参数）；再查执行顺序依赖（JUnit 支持随机顺序跑一次复现）；再查时区与 Locale 差异；最后在 CI 同款镜像里定位。CICD 篇的治理口径是 flaky 测试先 quarantine 并立单追踪，别直接重试掩盖。
## 8. 关联知识

- 前置：[Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
- 相关：[CI/CD](工程实践/运维/CICD.md)（流水线里跑测试与覆盖率门禁）、[Bean-Validation](开发语言/Java/Bean-Validation.md)（校验规则的测试落点）
- 延伸：Testcontainers、契约测试、变异测试（PIT）
