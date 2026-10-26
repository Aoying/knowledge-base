---
知识点: MyBatis-Plus
层级: 进阶
适用版本: MyBatis-Plus 3.5.3.2
更新日期: 2026-10-23
关联:
  - Java
  - Spring-Boot
标签:
  - Java
  - 实践
---

# MyBatis-Plus

## 1. 核心概念

MyBatis-Plus（MP）是在 MyBatis 之上做增强的工具层：不改 MyBatis 的 XML/注解 SQL 能力，额外提供通用 CRUD（`BaseMapper` + `IService`）、条件构造器（Wrapper）和插件体系（分页、乐观锁、防全表更新等拦截器）。定位判断：单表 CRUD 与简单条件查询用 MP 省样板代码；复杂多表关联、报表 SQL 仍写 XML，别硬用 Wrapper 拼——拼出来的链式调用可读性还不如 SQL。

实体映射要点：`@TableName` 指定表名（类名与表名不一致时必标）、`@TableField` 处理字段名不一致与「非表字段」（`exist = false`）；字段填充（创建时间、更新时间、创建人）用 `@TableField(fill = FieldFill.INSERT/INSERT_UPDATE)` + 全局 `MetaObjectHandler` 统一赋值，别在每个 Service 里手写 set。

## 2. 插件配置（新版写法）

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor i = new MybatisPlusInterceptor();
        i.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        i.addInnerInterceptor(new OptimisticLockerInnerInterceptor()); // 乐观锁
        return i;
    }
}
```

- 3.4+ 起旧的 `PaginationInterceptor` 已废弃，统一走 `MybatisPlusInterceptor` + 内部拦截器链；拦截器顺序有讲究：分页建议放最后添加（官方建议多租户等改写 SQL 的拦截器在前、分页在后）。
- 注意：PageHelper 和 MP 分页插件**二选一**，混用会出两套 limit。神农项目栈里 PageHelper 1.4.3 与 MP 并存，落点是：存量 XML 查询走 PageHelper，新代码走 MP 分页插件，同一条查询链路上只能出现一种，别叠加。
- OceanBase（MySQL 模式）分页方言按 `DbType.MYSQL` 处理即可（经验：3.5.x 无 OB 专属枚举时用 MYSQL 兼容）。
- 需要防误删/防全表更新时可再加 `BlockAttackInnerInterceptor`，它会拦截无 where 条件的 update/delete；批量维护数据的脚本型任务会被它拦下，评估后再开。

## 3. 条件构造器速记

```java
// lambda 写法，防字段名写错
lambdaQuery().eq(User::getAge, 18).like(User::getName, "张").list();
// 动态条件：第一个参数 boolean
lambdaQuery().eq(StringUtils.isNotBlank(name), User::getName, name);
// update 只更新非 null 字段；想把字段置 null 用 updateWrapper.set("col", null)
```

- 优先 `LambdaQueryWrapper/LambdaUpdateWrapper`：方法引用能在编译期发现字段改名，字符串版 Wrapper 的列名写错要到运行时才炸。
- 链式条件默认 AND；`or()` 切换下一段为 OR，嵌套条件用 `and(w -> ...)` / `or(w -> ...)` 括起来，别写出 `a AND b OR c` 这种靠 SQL 优先级猜的表达式。
- Wrapper 只在当前查询内有效，不要把 Wrapper 当参数在多层方法间传来传去，条件拼装散掉后没人能一眼看全这条 SQL。
- `list()` 无条件全表拉取是事故高发点：凡是前端可触发的列表查询，必须配分页或强制条件。

## 4. 分页用法

```java
// Service 层：Page 同时承载入参（当前页/每页数）与结果（total/records）
Page<Order> page = new Page<>(current, size);
orderService.page(page, Wrappers.<Order>lambdaQuery().eq(Order::getStatus, status));
// 物理分页由插件在 SQL 层拼 limit；注意 count 优化：复杂查询可关闭自动 count 后手写
```

分页对象传到 Controller 返回时，把 MP 的 `Page` 转成项目统一的分页响应结构，不要让前端直接依赖 MP 的 JSON 结构，升级时会被绑住（经验）。

## 5. 常见坑

1. **逻辑删除**：`@TableLogic` 字段，查询自动带 `deleted=0`；自定义 SQL 要自己拼条件。恢复数据、查已删数据的场景要绕开自动条件（手写 SQL 或临时方案），别改全局配置迁就个例。
2. **主键策略**：`@TableId(type = IdType.ASSIGN_ID)` 雪花算法；OceanBase 下自增主键行为跟 MySQL 有差异，优先用 ASSIGN_ID。AUTO（自增）只在单库简单场景用；INPUT 是自己赋值，导入历史数据时常用。
3. **`updateById` 忽略 null**：默认字段策略下 null 字段不更新，这是为防误清空；全量更新用 `update(entity, wrapper)`，或调字段策略 / 用 lambdaUpdate 的 `set` 显式赋值（含置 null）。
4. **乐观锁**：实体加 `@Version` 且注册乐观锁拦截器才生效；更新行数为 0 表示版本冲突，要给业务一个明确的冲突提示与重试策略，而不是笼统报「更新失败」。Flowable 的表不要乱加 MP 乐观锁，引擎自己管版本。
5. **大批量插入**：`saveBatch` 默认 1000 一批，调大 batchSize；超大批量考虑原生 JDBC batch。注意 MP 的 saveBatch 本质是攒批提交预编译语句，别期待它和 rewriteBatchedStatements 的驱动级批处理完全等价，百万级导入先小批量实测再定方案。
6. **枚举映射**：`@EnumValue` 标在存库的字段上，别标错；或实现 `IEnum`。前端要展示文案时另做转换，别把文案存进库。
7. **自动填充不生效**：`MetaObjectHandler` 里用 `strictInsertFill` 时字段名/类型对不上会静默跳过，新增填充字段后先写个单测验证真的填上了。

## 6. 与 Flowable 共存的注意点

神农项目里 MP 与 Flowable 6.7.2 同用一个数据源：Flowable 的表（ACT_*）由引擎自己的 MyBatis 管理，不要用 MP 的 BaseMapper 去套 ACT_* 表，也不要给引擎的表加逻辑删除/自动填充这类全局拦截行为；全局拦截器（如字段填充）要确认只作用于业务实体，`MetaObjectHandler` 对没有对应字段的对象不处理，一般无碍，但防全表更新插件这类 SQL 级拦截要评估对引擎 SQL 的影响。

## 实战案例（项目沉淀）

- [解抵押平台](项目沉淀/解抵押平台.md)：MyBatis-Plus 与 MySQL → Oracle SQL 迁移
- [神农-e贷-法人版](项目沉淀/神农-e贷-法人版.md)：企业级数据访问实战

## 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)
- [多数据源（dynamic-datasource）](开发语言/Java/多数据源.md)
- [Druid 监控与慢 SQL 定位](开发语言/Java/Druid.md)

- [MyBatis 原生](开发语言/Java/MyBatis/MyBatis原生.md)：Plus 覆盖不到的复杂映射、动态 SQL 与插件机制。
