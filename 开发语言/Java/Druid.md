---
知识点: Druid 监控与慢 SQL 定位
层级: 进阶
适用版本: Druid 1.2.x
更新日期: 2026-10-22
关联:
  - Java
  - MyBatis-Plus
  - MySQL-数据库
标签:
  - Java
  - 数据库
  - 实践
---

# Druid 监控与慢 SQL 定位

## Druid 监控与慢 SQL 定位

```yaml
# application.yml
spring:
  datasource:
    druid:
      filter:
        stat: { enabled: true, slow-sql-millis: 1000, log-slow-sql: true }
        wall: { enabled: true }   # 防火墙：防注入
```
```java
// 代码里拿慢 SQL TopN
DruidStatManagerFacade facade = DruidStatManagerFacade.getInstance();
List<Map<String,Object>> slow = facade.getDataSourceStatDataList(1).get(0)...;
```

- 浏览器开 `/druid/index.html` 看 SQL 监控页：执行次数、慢 SQL、 max 并发，一目了然（生产记得加账号密码）。
- 慢 SQL 定位链路：druid 慢日志 → `EXPLAIN` 看执行计划 → 索引/分页优化 → 回 druid 验证耗时下降。
- 神农这种金融项目：`slow-sql-millis` 设 500ms，定时扫监控页，比等用户投诉强。

