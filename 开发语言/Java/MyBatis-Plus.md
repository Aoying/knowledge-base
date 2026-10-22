---
知识点: MyBatis-Plus
层级: 进阶
适用版本: MyBatis-Plus 3.5.3.2
更新日期: 2026-10-22
关联:
  - Java
  - Spring-Boot
  - Java知识点地图
标签:
  - Java
  - 实践
---

# MyBatis-Plus

## MyBatis-Plus 3.5 实战

### 插件配置（新版写法，别再用旧的 PaginationInterceptor）
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
注意：PageHelper 和 MP 分页插件**二选一**，混用会出两套 limit。

### 条件构造器速记
```java
// lambda 写法，防字段名写错
lambdaQuery().eq(User::getAge, 18).like(User::getName, "张").list();
// 动态条件：第一个参数 boolean
lambdaQuery().eq(StringUtils.isNotBlank(name), User::getName, name);
// update 只更新非 null 字段；想把字段置 null 用 updateWrapper.set("col", null)
```

### 常见坑
1. **逻辑删除**：`@TableLogic` 字段，查询自动带 `deleted=0`；自定义 SQL 要自己拼条件。
2. **主键策略**：`@TableId(type = IdType.ASSIGN_ID)` 雪花算法；OceanBase 下自增主键行为跟 MySQL 有差异，优先用 ASSIGN_ID。
3. **`updateById` 忽略 null**：全量更新用 `update(entity, wrapper)` 或 lambdaUpdate。
4. **大批量插入**：`saveBatch` 默认 1000 一批，调大 batchSize；超大批量考虑原生 JDBC batch。
5. **枚举映射**：`@EnumValue` 标在存库的字段上，别标错。

