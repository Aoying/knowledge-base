---
知识点: Spring Cache 缓存抽象
层级: 进阶
适用版本: Spring Boot 2.7.3 / Spring Framework 5.3
更新日期: 2026-10-23
关联:
  - Spring-Boot
  - Redis
  - Redis-缓存问题与分布式锁
标签:
  - Java
  - 缓存
  - 实践
---

# Spring Cache 缓存抽象

## 1. 核心概念

Spring Cache 是**声明式缓存抽象**：用注解把「查缓存 → 回源 → 写缓存」的模板从业务代码里抽走，业务方法只管查数据库。存储可换：接口是 `CacheManager` / `Cache`，实现可以是本地 `ConcurrentHashMap`、Caffeine、Redis、EhCache，切换不动业务代码。

边界先说清：它只管**单次读写的缓存套路**，不管一致性深水区（双写、订阅 binlog）和穿透击穿雪崩的系统对策——那些在 [Redis 缓存问题与分布式锁](数据库/Redis-缓存问题与分布式锁.md) 里，本篇负责注解侧的正确用法。典型落点：字典码值、机构与产品参数、地区树等读多写少、容忍分钟级延迟的数据；强一致数据（余额、额度）不要套这层抽象。

## 2. 注解全家桶

入口：启动类或配置类加 `@EnableCaching`，不加所有注解静默无效（不报错，这是第一个坑）。

| 注解 | 时机 | 语义 |
| --- | --- | --- |
| `@Cacheable` | 调用前查 | 命中直接返回不进方法；未命中执行方法并回写 |
| `@CachePut` | 调用后写 | 方法必执行，结果覆盖缓存——用于更新方法 |
| `@CacheEvict` | 调用前/后删 | 删单个 key 或整表（`allEntries`），用于删除/失效方法 |
| `@Caching` | 组合 | 一个方法上叠多个上述注解 |
| `@CacheConfig` | 类级 | 统一本类默认 cacheNames、keyGenerator |

```java
@Service
@CacheConfig(cacheNames = "dict")
public class DictService {

    @Cacheable(key = "#type + ':' + #code")
    public DictItem getItem(String type, String code) {
        return dictMapper.selectItem(type, code); // 只有未命中才执行
    }

    @CachePut(key = "#item.type + ':' + #item.code")
    public DictItem updateItem(DictItem item) {
        dictMapper.updateItem(item);
        return item; // 返回值即写入缓存的内容，必须与查询返回结构一致
    }

    @CacheEvict(key = "#type + ':' + #code")
    public void deleteItem(String type, String code) {
        dictMapper.deleteItem(type, code);
    }
}
```

SpEL 关键参数：

- `key`：缓存键，可用参数 `#code`、结果 `#result`（仅 Put/Evict afterInvocation 或配合 unless）。
- `condition`：满足才缓存，如 `condition = "#code.length() > 0"`，在调用前求值。
- `unless`：结果满足条件则**不**缓存，如 `unless = "#result == null"` 不缓存空值——但注意不缓存空值等于放弃防穿透，取舍见第 5 节。

## 3. 切换到 Redis：CacheManager 配置

Boot 引入 `spring-boot-starter-data-redis` 后自动装配 `RedisCacheManager`，`spring.cache.type=redis` 可显式指定。默认配置基本不能直接上生产，必须改三处：序列化、TTL、空值策略。

```java
@Configuration
public class CacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // 每个缓存单独 TTL：字典类长、参数类短，不要全局一刀切
        Map<String, RedisCacheConfiguration> configs = new HashMap<>();
        configs.put("dict", baseConfig().entryTtl(Duration.ofHours(6)));
        configs.put("param", baseConfig().entryTtl(Duration.ofMinutes(30)));
        return RedisCacheManager.builder(factory)
                .cacheDefaults(baseConfig().entryTtl(Duration.ofMinutes(10)))
                .withInitialCacheConfigurations(configs)
                .build();
    }

    private RedisCacheConfiguration baseConfig() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .prefixCacheNameWith("cache:")          // key 带业务前缀，便于运维定位
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();            // 是否缓存空值，见第 5 节
    }
}
```

- 序列化别用默认 JDK 序列化：Redis 里是二进制不可读、类全名变更后反序列化直接失败；`GenericJackson2JsonRedisSerializer` 存 JSON 并带类型信息，运维可查、跨语言可读。
- TTL 必须设且各缓存错峰：无 TTL 等于缓存永不过期，数据改了永远读旧值；所有 key 同一 TTL 又会一起过期形成雪崩，TTL 分级 + 随机抖动的对策在 Redis 篇展开。

## 4. 一致性：注解侧能做的与不能做的

- 读路径是标准的 Cache Aside：`@Cacheable` 查不到才查库回写。写路径的纪律：**先更新数据库，再删缓存**（`@CacheEvict` 放在写方法上），不要用 `@CachePut` 去「同步更新缓存」当一致性手段——高并发下两个写请求交错时 Put 的顺序可能与 DB 提交顺序相反，缓存里反而留下旧值。`@CachePut` 只适合单写、低并发的配置型数据。
- **与事务的顺序坑**：`@CacheEvict` 默认在方法返回后执行，但如果方法在事务里，缓存删除发生的时点可能早于事务提交——删完到提交之间有读请求进来，又把旧值回写进缓存。两种解法：把 evict 动作注册到事务提交后（`TransactionSynchronizationManager` 的 `afterCommit` 里执行），或给缓存切面与事务切面排 order 并接受最终一致。做不到这两点就老实接受「短暂不一致 + TTL 兜底」，并在选型时就排除强一致数据。
- **自调用失效**：同类内 `this.getItem()` 不走 AOP 代理，`@Cacheable` 静默不生效——和 `@Transactional` 自调用是同一个病根，解法同样是注入自身代理或拆类，见 [Spring 事务](开发语言/Java/Spring-事务.md)。

## 5. 疑难：与穿透/击穿的衔接

- **穿透**（查不存在的数据反复打库）：注解侧落法是缓存空值——去掉 `disableCachingNullValues` 与 `unless = "#result == null"`，并给这类缓存设短 TTL（如 1 分钟）。空值开关要按缓存分别配置，别一个开关管全部。
- **击穿**（热点 key 过期瞬间并发回源）：`@Cacheable(sync = true)` 只保证同一 JVM 内同 key 单线程回源，**跨实例无效**——集群下仍靠分布式锁或逻辑过期，见 Redis 篇。
- **雪崩**：本篇能做的只有 TTL 分级，随机过期与多级缓存策略在 Redis 篇。
- 本地缓存（ConcurrentHashMap 实现）的引用污染：存的是对象引用，调用方改返回值等于改缓存；Redis 实现因序列化存副本，没这个问题。混用时注意这层差异。

## 6. 常见误区

- 误区：加了注解缓存就生效了。自调用、私有方法、没加 `@EnableCaching`、类没被 Spring 管理，四种静默失效都不报错，排查先按这张单子过。
- 误区：`@CacheEvict(allEntries = true)` 随手用在更新方法上。它清的是整个 cache 空间，Redis 实现下按前缀批量删除，大缓存空间会阻塞；常规更新只删相关 key。
- 误区：key 用默认生成（全部参数拼接）就够了。参数是复杂对象时默认 key 靠 `equals/hashCode`，对象没重写就会每次生成新 key、缓存永不命中；复杂参数显式写 `key` 或自定义 `KeyGenerator`。
- 误区：把缓存当数据源做分页、聚合查询。缓存只适合按 key 直取，复杂查询回数据库或搜索引擎。

## 7. 版本与延伸

- 本地高性能需求可把 CacheManager 换成 Caffeine（`spring-boot-starter-cache` + caffeine 依赖，`spring.cache.type=caffeine`），注解不变——这正是抽象层的价值。
- 多级缓存（一级 Caffeine 抗热点、二级 Redis 跨实例共享）没有开箱即用的官方实现，需要自定义 CacheManager 或引入专用组件，且要解决一级缓存的跨实例失效广播问题，不要低估复杂度，无明确瓶颈前不上。

## 8. 面试与自测

- `@Cacheable` 与 `@CachePut` 的执行时机与返回值语义差在哪？
- 为什么写路径推荐删缓存而不是更新缓存？
- `@Cacheable` 自调用为什么不生效？与事务自调用什么关系？
- `sync = true` 的防击穿范围到哪一层为止？
- Redis 实现下哪三个默认配置必须改？为什么？

## 9. 关联知识

- [Spring Boot](开发语言/Java/Spring-Boot.md)：自动配置与 starter，缓存与 Redis 的装配入口。
- [Redis](数据库/Redis.md)：底层存储本身的数据结构与使用。
- [Redis 缓存问题与分布式锁](数据库/Redis-缓存问题与分布式锁.md)：穿透/击穿/雪崩与一致性的系统级对策，与本篇的注解落法对应。
- [Spring 事务](开发语言/Java/Spring-事务.md)：代理自调用与切面顺序的同源问题。
