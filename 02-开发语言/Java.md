# Java

> 每晚 20:30 自动补充完善。贴合 Spring Boot 2.7.x + MyBatis-Plus 3.5.x 技术栈（神农 e 贷法人版同款）。

## Spring Boot 2.7 实战

### 配置加载顺序（后覆盖前）
1. 命令行参数 `--server.port=8081`
2. `SPRING_APPLICATION_JSON` / 系统环境变量
3. `application-{profile}.yml` → `application.yml`

### 常用配置片段
```yaml
server:
  port: 8080
  tomcat:
    max-threads: 200
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initial-size: 5
      max-active: 50
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 本地看 SQL
  global-config:
    db-config:
      logic-delete-value: 1
      logic-not-delete-value: 0
```

### @Transactional 失效的 4 个经典场景
1. **同类自调用**：`this` 直接调，绕过代理。解法：注入自己 / `AopContext.currentProxy()`。
2. **private / final 方法**：代理不了，不生效。
3. **异常被吞**：try-catch 吃掉异常且没抛，事务以为成功。想回滚就 `throw new RuntimeException` 或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
4. **checked 异常**：默认只回滚 RuntimeException。受检异常要回滚加 `rollbackFor = Exception.class`。

### 事务传播常用组合
- `REQUIRED`（默认）：有就加入，没有新建。
- `REQUIRES_NEW`：挂起当前，永远新建——记流水、审计日志独立提交用它。
- 注意：`REQUIRES_NEW` 内部抛异常不影响外部已提交的事务（已挂起隔离）。

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

## 排查三板斧

1. **SQL 不对**：本地开 `log-impl: stdout`，直接看最终 SQL 和参数。
2. **N+1**：循环里调 mapper → 改成一次 `in` 查询 + 内存组装。
3. **慢接口**：先看 SQL 执行计划（`EXPLAIN`），再看是不是事务里套了远程调用（事务+Feign 长时间占用连接是大忌）。

## 待补充
- Flowable 6.7 实战（见 flowable/ 方向）
- Druid 监控与慢 SQL 定位
- 本地多数据源 dynamic-datasource 笔记

## 疑难杂症

### 1. OceanBase 下大事务超时回滚，MySQL 上好好的
- **现象**：批量更新几万行，MySQL 正常，OceanBase 报事务超时回滚。
- **原因**：OceanBase 对大事务限制更严（undo 保留、超时阈值），默认超时参数和 MySQL 不同。
- **解决**：大批量拆小批次提交；调大 `ob_trx_timeout`；能异步就别放事务里。

### 2. 内层 try-catch 吃掉异常，外层事务不回滚
- **现象**：`@Transactional` 方法里 try-catch 了内层异常，业务失败了但数据已提交。
- **原因**：Spring 靠异常触发回滚；异常被 catch 后事务拦截器认为"正常返回"直接提交。
- **解决**：catch 后 `throw new RuntimeException(e)` 重新抛出，或手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。

### 3. Druid 连接"假死"：能连上但 SQL 一直 hang
- **现象**：服务跑着跑着部分请求卡死，重启就好。
- **原因**：数据库端杀掉了空闲连接（如 wait_timeout），但连接池不知道，拿到的是死连接。
- **解决**：Druid 开 `testWhileIdle=true` + `validationQuery=SELECT 1`，`timeBetweenEvictionRunsMillis` 小于数据库的 wait_timeout。
