---
知识点: MyBatis 原生
层级: 基础
适用版本: MyBatis 3.5 / Spring Boot 2.7
更新日期: 2026-10-24
关联:
  - MyBatis-Plus
  - Java
  - Druid
  - 多数据源
  - MySQL 数据库
标签:
  - Java
  - 数据库
  - 实践
---

# MyBatis 原生

## 1. 核心概念

MyBatis 是半自动 ORM：SQL 写在 XML 或注解里，框架只负责参数绑定、结果映射和执行调度（`SqlSession` → `Executor` → `StatementHandler` → JDBC），不生成 SQL，复杂 SQL 的可控性是它的全部价值。

与 [MyBatis-Plus](开发语言/Java/MyBatis-Plus.md) 的分工：**Plus 是增强，不是替代**。`BaseMapper`、Wrapper 都建在原生能力之上，动态 SQL、resultMap、插件机制绕不开。落地口径：单表 CRUD 和简单条件走 Plus；多表关联、报表统计写 XML 原生 SQL，不要用 Wrapper 硬拼长链。

## 2. 基础知识

**XML 与注解取舍**：注解适合短 SQL（按主键查、简单更新）；SQL 超过几行、有动态片段或嵌套映射，一律 XML——注解里拼多行字符串不可读，也不能复用 `<sql>` 片段。项目惯例：Mapper 接口与 XML 同名，`namespace` 是接口全限定名，`id` 与方法名一一对应，写错要到运行时才报 `BindingException`，改名两边一起改。

```xml
<mapper namespace="com.example.loan.mapper.LoanApplyMapper">
    <select id="selectById" resultType="com.example.loan.entity.LoanApply">
        SELECT id, cust_name, amount, status FROM loan_apply WHERE id = #{id}
    </select>
</mapper>
```

**`#{}` 与 `${}`**：`#{}` 是预编译占位符，走 `PreparedStatement` 设值，天然防注入，**默认永远用它**；`${}` 是字符串直接拼接，只允许用于表名、列名、排序字段这类不能用占位符的位置，且值必须经枚举/白名单校验——`${}` 拼用户输入就是教科书级 SQL 注入。

## 3. 进阶知识

**动态 SQL** 常用五件套：

```xml
<select id="search" resultMap="applyMap">
    SELECT * FROM loan_apply
    <where>
        <if test="custName != null and custName != ''">
            AND cust_name LIKE CONCAT('%', #{custName}, '%')
        </if>
        <choose>
            <when test="status != null">AND status = #{status}</when>
            <otherwise>AND status != 'DELETED'</otherwise>
        </choose>
    </where>
    ORDER BY create_time DESC
</select>

<update id="updateSelective">
    UPDATE loan_apply
    <set>
        <if test="amount != null">amount = #{amount},</if>
        <if test="status != null">status = #{status},</if>
    </set>
    WHERE id = #{id}
</update>

<!-- 批量插入 -->
<insert id="insertBatch">
    INSERT INTO loan_apply (cust_name, amount, status) VALUES
    <foreach collection="list" item="it" separator=",">
        (#{it.custName}, #{it.amount}, #{it.status})
    </foreach>
</insert>
```

要点：`<where>` 自动去开头多余的 AND/OR，`<set>` 自动去结尾逗号，`<trim>` 是两者的通用版，自定义拼接用它。`test` 里是 OGNL 表达式，字符串比较用 `==`，别写成单个 `=`。多参数方法用 `@Param` 命名后按名引用，不要靠 `param1/arg0`。

**resultMap 与嵌套映射**：列名与属性名不一致、或一对一/一对多关联时用 resultMap，不要指望自动映射硬凑：

```xml
<resultMap id="applyMap" type="com.example.loan.vo.ApplyVO">
    <id property="id" column="id"/>
    <result property="custName" column="cust_name"/>
    <!-- 一对一 -->
    <association property="customer" javaType="com.example.loan.entity.Customer">
        <id property="id" column="cust_id"/>
        <result property="name" column="cust_name"/>
    </association>
    <!-- 一对多：主表必须用 <id> 标注 -->
    <collection property="files" ofType="com.example.loan.entity.ApplyFile">
        <id property="id" column="file_id"/>
        <result property="fileName" column="file_name"/>
    </collection>
</resultMap>
```

`association/collection` 两种装配方式：嵌套结果（上面这种，一条 JOIN 搞定）和嵌套查询（`select` 属性指向另一条语句，产生 N+1 查询）。默认嵌套结果；嵌套查询只适合关联量小、配合懒加载按需取的场景。

**懒加载**：`lazyLoadingEnabled=true` 后嵌套查询属性在访问时才发 SQL。坑：序列化/日志打印/JSON 返回都会触发「访问」，看似懒加载实际全量发出；脱离 SqlSession 后访问还会报错。结论：只在明确按需场景开，不要全局打开当优化手段。

## 4. 原理剖析

**插件/拦截器机制**：MyBatis 只允许拦截四个对象——`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`。插件用 `@Intercepts` + `@Signature` 声明拦截点，框架对这四个对象建动态代理，多个插件按注册顺序层层包装成责任链。

**PageHelper 本质就是一个 MyBatis 插件**：它拦截 `Executor` 的 query，在执行前根据 `PageHelper.startPage()` 放进 ThreadLocal 的分页参数改写 SQL（拼 limit）并先跑一条 count 查询。由此可推两个现象：`startPage` 后紧跟的第一条查询才被分页，中间隔了别的查询就分错；异常路径下 ThreadLocal 没清掉会污染同线程下一次查询。PageHelper 与 Plus 分页拦截器二选一，混用会拼两套 limit。

自写插件的典型用途是数据权限（自动追加机构条件）、SQL 审计；只拦截需要的方法，`intercept` 改完参数必须 `invocation.proceed()`，吞掉等于把原执行短路掉。

**Executor 三种类型**：`SIMPLE`（默认，每次新建 Statement）、`REUSE`（复用 Statement）、`BATCH`（攒批，`commit` 或 `flushStatements` 时才真正发送，期间拿不到自增主键）。`BATCH` 只攒不发，批量代码写完发现库里没数据，多半是没提交会话。

## 5. 实战应用

批量插入按量级选路：几千条用 `foreach` 多值 VALUES 最简单；十万级以上用 BATCH Executor 或 JDBC 原生 batch，并打开连接参数 `rewriteBatchedStatements=true`（在 [Druid](开发语言/Java/Druid.md) 连接串上配），不开驱动会逐条发送，攒批等于白攒。手写 BATCH 每 1000 条 flush + 提交一段，别一口气攒百万条。

```java
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {
    LoanApplyMapper mapper = session.getMapper(LoanApplyMapper.class);
    for (int i = 0; i < list.size(); i++) {
        mapper.insertOne(list.get(i));
        if (i % 1000 == 999) { session.flushStatements(); session.commit(); }
    }
    session.commit();
}
```

多数据源场景下 Mapper 与 SqlSessionFactory 的绑定见 [多数据源](开发语言/Java/多数据源.md)：扫描路径配错，SQL 会打到另一套库且不报错、只查不到数据，排查先看日志里的数据源名。

## 6. 疑难杂症

- **改了 XML 不重启不生效，还报旧错**：XML 是启动时解析进 `Configuration` 的，改完必须重启，别对着新文件查旧堆栈。
- **二级缓存查出脏数据**：二级缓存按 namespace 跨 SqlSession 共享，但**关联表被别的 namespace 更新时，本 namespace 的缓存不会失效**，这是官方明确的局限——实务结论：二级缓存默认不开，缓存交给 Redis。一级缓存（SqlSession 级，默认开）的坑是：别的会话/直连改了库，本会话仍读旧值，长会话里查列表要留意。
- **foreach 批量插入报包过大**：多值 VALUES 拼出的 SQL 超过 `max_allowed_packet`，分片提交（每 500 条一刀），不要调大参数硬扛。
- **`${}` 排序注入**：前端传的 `orderBy` 直接 `${orderBy}` 拼接被注入。解法：排序字段/方向后端用枚举白名单映射，前端只传枚举 key。
- **OceanBase（MySQL 模式）注意**：语法方言按 MySQL 写基本兼容；个别特性与自增/序列行为有差异，批量插入与主键策略在预发实测过再上生产。

## 7. 常见误区

- 以为用了 Plus 就不用懂原生：Wrapper 排不了的 SQL 最终都要回 XML。
- `resultType` 偷懒用 `Map`：改列名静默丢字段；正式代码用实体/VO + resultMap，或开下划线转驼峰（`mapUnderscoreToCamelCase=true`）。
- 嵌套查询（`select` 式 association）无脑用：一对多列表页一次请求打出上百条 SQL，先数日志里的 SQL 条数再谈慢。
- 批量场景还逐行循环调 `insert`：每行一次往返，慢是必然的。

## 8. 性能优化

- SQL 慢先看执行计划与索引，口径见 [MySQL 数据库](数据库/MySQL-数据库.md)；MyBatis 层能做的只有：不查多余列（忌 `SELECT *` 带出大字段）、嵌套结果一次 JOIN 代替 N+1、批量写走批处理。
- `logImpl` 打 SQL 日志只放开发/预发：生产全量打会把日志盘打爆，参数里还可能有敏感信息。
- 一级缓存只对同一会话重复查询有效，跨请求缓存走 Redis，别指望它。

## 9. 版本差异

- 本项目栈：mybatis-spring-boot-starter 2.2.2（MyBatis 3.5.9 线）+ MyBatis-Plus 3.5.3.2 + PageHelper 1.4.3，跑在 Spring Boot 2.7.3 上，三者版本成套看，别单独升大版本。
- MyBatis 3.5.x：默认 `aggressiveLazyLoading` 已改为 false；其余小增补不影响存量 XML 写法。
- PageHelper 的 API（`startPage`/`PageInfo`）在 1.4→2.x 间有变化，升级前先核对分页调用点。

## 10. 面试与自测

- `#{}` 和 `${}` 的区别是什么？`${}` 什么时候不得不用，怎么防注入？
- PageHelper 的分页原理是什么？它为什么能「拦截」到你的查询？
- 一级缓存和二级缓存的作用域分别是什么？二级缓存为什么不建议开？
- association 用嵌套查询写一对多会有什么问题？怎么改成嵌套结果？
- BATCH Executor 为什么执行了 insert 却查不到数据？

### 参考答案

1. `#{}` 是预编译占位符，走 `PreparedStatement` 设值，天然防注入，默认永远用它；`${}` 是字符串直接拼接，不经过预编译。`${}` 只允许用于表名、列名、排序字段这类不能用占位符的位置，且值必须经枚举/白名单校验；把用户输入直接 `${}` 拼接就是教科书级 SQL 注入，如排序字段应由后端用枚举白名单映射，前端只传枚举 key。
2. PageHelper 本质是一个 MyBatis 插件：MyBatis 只允许拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler 四个对象，插件用 `@Intercepts` + `@Signature` 声明拦截点，框架对这四个对象建动态代理、多个插件按注册顺序层层包装成责任链。PageHelper 拦截 Executor 的 query，读取 `PageHelper.startPage()` 放进 ThreadLocal 的分页参数，改写 SQL 拼 limit 并先跑一条 count 查询。由此可推：startPage 后紧跟的第一条查询才被分页，中间隔了别的查询就分错；异常路径下 ThreadLocal 没清掉会污染同线程下一次查询；它与 Plus 分页拦截器二选一，混用会拼两套 limit。
3. 一级缓存作用域是 SqlSession，默认开启，只对同一会话内的重复查询有效；二级缓存作用域是 namespace，跨 SqlSession 共享。二级缓存不建议开，因为关联表被别的 namespace 更新时，本 namespace 的缓存不会失效，这是官方明确的局限，会查出脏数据；实务结论是二级缓存默认不开，缓存交给 Redis。
4. association 用嵌套查询写一对多会产生 N+1 查询：先查主表 1 条，再为每个主对象各发一条关联查询，一对多列表页一次请求可能打出上百条 SQL；序列化、日志打印等「访问」还会触发懒加载，雪上加霜。应改成嵌套结果：一条 JOIN 一次查出主从数据，在 resultMap 里用 `<collection>` 做一对多映射，且主表必须用 `<id>` 标注，主键相同的行才会被合并成一个主对象下的集合。
5. 因为 BATCH Executor 只攒不发：它把语句攒起来，到 `commit` 或 `flushStatements` 时才真正发送，期间还拿不到自增主键。执行了 insert 却查不到数据，多半是会话没提交；写完要 `session.commit()`，大批量按正文惯例每 1000 条 flush + 提交一段，别一口气攒百万条，也别忘了批量连接参数 `rewriteBatchedStatements=true`，否则驱动会逐条发送、攒批等于白攒。

## 11. 关联知识

- [MyBatis-Plus](开发语言/Java/MyBatis-Plus.md)：增强层，与本篇是分工关系
- [Druid](开发语言/Java/Druid.md)：连接池、慢 SQL 定位与批处理连接参数
- [多数据源](开发语言/Java/多数据源.md)：Mapper 与 SqlSessionFactory 的绑定
- [MySQL 数据库](数据库/MySQL-数据库.md)：SQL 与索引基本功
