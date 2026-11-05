---
知识点: Spring Cache 缓存抽象
层级: 进阶
适用版本: Spring Boot 2.7.3 / Spring Framework 5.3
更新日期: 2026-11-05
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

边界先说清：它只管**单次读写的缓存套路**，不管一致性深水区（双写、订阅 binlog）和穿透击穿雪崩的系统对策——那些在 [Redis 缓存问题与分布式锁](数据库/Redis/Redis-缓存问题与分布式锁.md) 里，本篇负责注解侧的正确用法。典型落点：字典码值、机构与产品参数、地区树等读多写少、容忍分钟级延迟的数据；强一致数据（余额、额度）不要套这层抽象。

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
- **自调用失效**：同类内 `this.getItem()` 不走 AOP 代理，`@Cacheable` 静默不生效——和 `@Transactional` 自调用是同一个病根，解法同样是注入自身代理或拆类，见 [Spring 事务](开发语言/Java/Spring/Spring-事务.md)。

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

## 7. 本地缓存 Caffeine 的调参与监控

Caffeine 是进程内高性能缓存：无网络往返，适合字典、参数这类读多写少的热点数据扛在 JVM 里。第 3 节的 Redis 是跨实例共享的远端缓存，两者定位不同——Caffeine 快但只在单实例内有效，跨实例一致性要另想办法（见本节末尾）。

### 7.1 接入方式

加两个依赖，注解不用动：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```properties
spring.cache.type=caffeine
spring.cache.cache-names=dict,param
spring.cache.caffeine.spec=maximumSize=10000,expireAfterWrite=10m,recordStats
```

- 依赖在 classpath 上时，Boot 自动装配从 `ConcurrentMapCacheManager` 切换为 `CaffeineCacheManager`；`spring.cache.type=caffeine` 是显式指定，写不写效果一样，但写了意图更清楚。
- `spring.cache.caffeine.spec` 是一串 Caffeine 规约字符串，逗号分隔；它对 `cache-names` 里列出的所有缓存一刀切——各缓存要不同策略时不能只靠 spec，改自定义 `CaffeineCacheManager` Bean（`setCaffeine` / `setCacheSpecification` 分别配），见 7.4。
- 版本号不用手写：caffeine 版本由 Spring Boot 依赖管理统一控制，跟 Boot 走即可。

### 7.2 关键调参：四个参数的语义差异

| 参数 | 计时起点 | 到期后行为 |
| --- | --- | --- |
| `maximumSize` | — | 超过条目上限时按淘汰策略挤出旧条目，约束内存 |
| `expireAfterWrite` | 写入 / 最近一次替换值 | 到点即过期，下次访问重新回源（写后固定时长，读多少次都不续命） |
| `expireAfterAccess` | 最近一次读或写 | 闲置到点才过期，持续被访问的热点 key 永不过期 |
| `refreshAfterWrite` | 写入 / 最近一次替换值 | 到点变「可刷新」而非过期：首次访问到旧条目时异步刷新并立即返回旧值 |

- `expireAfterWrite` 与 `expireAfterAccess` 的选型看数据性质：字典、参数这类要求「最多旧 10 分钟」的数据用 `expireAfterWrite`；会话、临时令牌这类「闲置才失效」的数据用 `expireAfterAccess`。两者同时配时以谁为准，待验证（只见到非官方资料称 `expireAfterWrite` 优先，未在官方文档中核实）。
- 过期是惰性淘汰：过期条目在下次访问、写操作或 `cleanUp()` 触发的维护时才被清理，但过期后读操作永远看不到它；不要指望到点准时从内存消失。
- `refreshAfterWrite` 最容易配错：它**不是过期**，只对 `LoadingCache` 生效，刷新走 `CacheLoader.reload(K, V)`；触发刷新的那次请求异步调 `reload`、立即返回旧值，刷新期间的并发请求也都读旧值；刷新抛异常会被记录后吞掉，旧值继续留着。**首次加载仍是同步的**：缓存里没有该 key 时，`@Cacheable` 方法照常同步执行并回写，`refreshAfterWrite` 管不着第一次。
- 关键坑：Spring 默认的 `CaffeineCacheManager`（只配 spec、没配 loader）建的是普通 `Cache`，不是 `LoadingCache`——只在 spec 里写 `refreshAfterWrite`、不配 `CacheLoader`，刷新永远不会发生。想用它要显式调 `CaffeineCacheManager.setCacheLoader(...)` 把缓存变成 `LoadingCache`；`@Cacheable` 的方法体仍是首次加载的回源逻辑，`CacheLoader` 只负责后续的异步刷新。

### 7.3 淘汰策略：近似 LRU（Window TinyLFU）

一句话原理：新条目先进一个很小的 LRU 窗口（只看新近性），窗口挤出的受害者由 TinyLFU 用 Count-Min Sketch 估算历史访问频率来裁决——频率高于主缓存受害者才准入主缓存（分段 LRU），否则直接丢弃；既留得住刚来的热点，也防得住一波流的冷数据把高频旧条目挤走。

推论（非官方原文，凭上述机制得出）：`maximumSize` 设多小直接决定这套策略的腾挪空间——设得太小，窗口和主缓存都装不下工作集，命中率上不去；调参没有通用最优值，看 `recordStats` 的命中率说话。

### 7.4 recordStats 开启与命中率监控

默认不统计，必须显式开：spec 里加 `recordStats`，或编程式 `Caffeine.newBuilder().recordStats()`。

```java
// 临时排查：直接读统计快照
Cache dict = (Cache) cacheManager.getCache("dict").getNativeCache();
CacheStats stats = dict.stats();
double hitRate = stats.hitRate();          // 命中率
long evictions = stats.evictionCount();    // 淘汰条数
long misses = stats.missCount();
```

- `Cache.stats()` 返回 `hitCount / missCount / hitRate / evictionCount / loadSuccessCount / averageLoadPenalty` 等快照；没开 `recordStats` 时这些计数全是 0，别拿 0 当「缓存没命中」去调参。
- 接入 Micrometer：`CaffeineCacheMetrics.monitor(registry, cache, "dict")`，同样要求先 `recordStats()`，否则绑上去的指标恒为 0。Spring Boot Actuator 会在启动时自动给已配置的缓存绑 `cache.*` 指标（`cache.gets`、`cache.evictions` 等），`/actuator/caches` 能看缓存清单；但启动后动态新建的缓存不会自动绑，要用 `CacheMetricsRegistrar` 手动注册。
- 调参闭环：先开 `recordStats` 跑一段时间，看命中率与淘汰数再动 `maximumSize` 与过期时间；命中率低先分清是容量不够（淘汰数高）还是过期太短（miss 高但淘汰低），两种病因的药方不一样。

### 7.5 与 Redis 二级缓存混用时的一级失效问题（点到为止）

Caffeine 做一级、Redis 做二级时，一级缓存是每个 JVM 各一份的独立副本：本实例 `@CacheEvict` 只清掉本机的一级缓存，其它实例的一级缓存还留着旧值，直到各自过期。跨实例失效要靠失效广播（如 Redis 发布订阅）或接受「一级缓存 TTL 内各实例读到的值可能不一致」，不要默认「删了一处就等于全清了」。多级缓存本身没有开箱即用的官方实现，见 §8。

### 7.6 边界条件与易错点

1. **不设 `maximumSize` 等于无界增长**：Caffeine 默认不做任何淘汰，不配上限的缓存会随条目增长一直占内存；本地缓存的第一行配置就该是容量上限，过期时间不能代替它（过期只解决旧数据，不解决条目总数）。
2. **只在 spec 里写 `refreshAfterWrite` 不配 `CacheLoader`，刷新不生效**：见 7.2，`CaffeineCacheManager` 默认建普通 `Cache`；发现「到期后还是旧值、也没见刷新」时先查有没有 `setCacheLoader`，别先怀疑过期时间写错了。
3. **`expireAfterAccess` 会让热点 key 永不过期**：持续被访问的 key 计时器不断重置，配了它就等于放弃了「定时刷新」；要求数据最多旧 N 分钟的场景必须用 `expireAfterWrite`，两者别按「都是过期」混着配。
4. `maximumSize` 与 `maximumWeight` 不能同时用，二选一；按条目数控用前者，按字节/权重控用后者（需另配 `weigher`）。

来源（核验日期 2026-11-05）：Caffeine 官方 Javadoc（过期/刷新语义、recordStats）；Spring Framework 5.3.32 `CaffeineCacheManager` Javadoc（`setCacheLoader` 建 `LoadingCache`、`setCaffeine` / `setCacheSpecification`）；Caffeine 作者 Ben Manes 的 W-TinyLFU 说明与 TinyLFU 论文（Einziger 等，ACM Trans. Storage 2017）；Spring Boot 参考文档（Actuator 缓存指标与 `CacheMetricsRegistrar`）。`expireAfterWrite` 与 `expireAfterAccess` 同时配置时的优先关系待验证，未作肯定表述。

## 8. 版本与延伸

- 本地高性能需求可把 CacheManager 换成 Caffeine（`spring-boot-starter-cache` + caffeine 依赖，`spring.cache.type=caffeine`），注解不变——这正是抽象层的价值。
- 多级缓存（一级 Caffeine 抗热点、二级 Redis 跨实例共享）没有开箱即用的官方实现，需要自定义 CacheManager 或引入专用组件，且要解决一级缓存的跨实例失效广播问题，不要低估复杂度，无明确瓶颈前不上。

## 小技巧与冷知识

【小技巧】`condition` 和 `unless` 的求值时点不一样，别混着用：`condition` 在方法调用前求值，决定“这次查不查缓存”；`unless` 在方法执行完后求值，决定“这次的结果写不写缓存”。两者可以叠加，比如 `condition = "#id != null"` 拦住空参数不查缓存，再配 `unless = "#result == null"` 让空结果不进缓存；只写一个，另一半的口子就还开着。

【小技巧】`@CacheEvict` 默认是方法成功返回后才删缓存，方法抛异常就不删；想“不管成败都删”才加 `beforeInvocation = true`。删除方法本身失败时默认不删是对的——库里没删掉、缓存也留着，下次读到的还是旧值而不是“库里有、缓存没了”的空洞；别顺手把 `beforeInvocation = true` 当默认值写。

【小技巧】`@Cacheable` 写多个 cacheNames 时按定义顺序逐个查，命中一个就直接返回，没命中的那些也会被补写一份。迁移缓存空间时可以利用这点：`@Cacheable({"dictNew", "dictOld"})`，读时先查新空间、再查旧空间兜底，方法执行一次、两个空间都写上，旧空间自然就被新数据覆盖完了。

【小技巧】key 表达式里能用 `#root` 变量，不用每个方法手写前缀防冲突：`key = "#root.methodName + ':' + #id"`，key 天生带上方法名。`#root.target` 是被代理的目标对象、`#root.caches` 是本次操作涉及的缓存集合，拼复杂 key 时比只拿参数名灵活；原文的 SpEL 一节只列了参数和结果，这几个根变量是同一套表达式里的。

【小技巧】多个方法共用一套 key 规则时，抽一个 `KeyGenerator` Bean：`keyGenerator = "bizKeyGenerator"`，所有方法共用同一套拼 key 的逻辑，改规则只改一处。注意 `key` 和 `keyGenerator` 不能同时写，两个都写了启动时就抛异常；二选一，固定规则用 keyGenerator、一次性拼写用 key。

【小技巧】`sync = true` 除了原文说的跨实例无效，还有三条硬限制：不能配 `unless`、只能写一个 cache、不能和其他缓存操作组合。违反时不是静默失效，而是启动或调用时直接报错；热点 key 防击穿只用 sync 的前提是单实例、单缓存、不配 unless，三个条件缺一个就别硬上。

【小技巧】不走注解也能直接操作缓存：注入 `CacheManager`，`cacheManager.getCache("dict").evict(key)` 删一条、`clear()` 清整个空间，管理端的“一键清缓存”接口、测试里清场都用它。再往下还有 `getNativeCache()`，能拿到底层的原生对象（Redis 的 `RedisTemplate`、Caffeine 的 `Cache`），注解够不着的精细操作走这条路。

【小技巧】Redis 里真实 key 的拼法是 `前缀 + 缓存名 + "::" + key`：原文配了 `prefixCacheNameWith("cache:")`，dict 缓存的真实 key 就是 `cache:dict::type:code` 这种形式。用 redis-cli 排查时按这个完整格式搜，别只搜业务 key；不同缓存空间同名 key 也不会串，就是因为中间隔着缓存名这一段。

【小技巧】`@Cacheable` 标在类上对全类方法都生效，配合 `@CacheConfig` 统一 cacheNames，整个 Service 不用每个方法重复写缓存名；某个方法想例外时再在方法上单独标注解，方法级优先于类级。原文只讲了类级 `@CacheConfig`，类级 `@Cacheable` 是同一层面的省事写法。

【冷知识】Spring 的缓存抽象是 3.1 版本（2011 年）引入的，比 Spring Boot 还早两年多；最初开箱只有 ConcurrentMap 和 EhCache 两种实现，Redis、Caffeine 都是后来才接进来的。所以“缓存抽象”本身不绑定任何存储，原文第 3 节换 Redis 只改配置不动业务代码，正是这层抽象从第一天起的设计目标。

【冷知识】`GenericJackson2JsonRedisSerializer` 存进 Redis 的 JSON 里带 `@class` 字段，那是 Jackson default typing 写进去的类型标记，反序列化靠它认出原来是哪个类。用 redis-cli 直接改缓存值时别删 `@class`，删了就认不出类型；看到值里混着 `@class` 也别当成业务字段去解析，它是序列化器的记号。

【小技巧】单元测试不用起 Redis：配一个 `ConcurrentMapCacheManager` 的 Bean，注解逻辑照常跑，断言“第二次调用不进方法”就能验证缓存生效。切 Redis 只是换一个 `CacheManager` 实现，业务代码和注解一行不用动；等联调时再换 RedisCacheManager 去验序列化和 TTL。

【小技巧】更新一个数据要清多个缓存空间时，用 `@Caching` 一次写完：`@Caching(evict = {@CacheEvict(value = "dict", key = "#type"), @CacheEvict(value = "param", allEntries = true)})`，一个方法上叠两个 evict。原文的注解表只列了 `@Caching` 是组合注解，这个“一次更新、多处失效”就是它最常见的用法，别拆成两个方法各清一处。

【冷知识】`sync` 属性是 Spring 4.3 才加上的，原文适用版本 5.3 已经有了；从 4.2 及更早版本升上来的老项目，直接抄 `sync = true` 会编译不过。顺带一提，缓存抽象本身从 3.1 到 4.1 还补过一次大升级（JSR-107 注解支持），看老资料时注意版本，别把新属性往旧版本上套。

## 9. 面试与自测

- `@Cacheable` 与 `@CachePut` 的执行时机与返回值语义差在哪？
- 为什么写路径推荐删缓存而不是更新缓存？
- `@Cacheable` 自调用为什么不生效？与事务自调用什么关系？
- `sync = true` 的防击穿范围到哪一层为止？
- Redis 实现下哪三个默认配置必须改？为什么？

- 只在 `spring.cache.caffeine.spec` 里写了 `refreshAfterWrite=10m`，为什么缓存到期后没有自动刷新？`refreshAfterWrite` 与 `expireAfterWrite` 的行为差在哪？

### 参考答案

1. @Cacheable 在调用前查缓存：命中直接返回、不进方法，未命中才执行方法并把结果回写。@CachePut 在调用后写缓存：方法必执行，结果覆盖缓存，用于更新方法，且返回值即写入缓存的内容，必须与查询返回结构一致。简言之，前者是「有则不查」，后者是「必查必写」。
2. 因为高并发下两个写请求交错时，@CachePut 写缓存的顺序可能与数据库提交顺序相反，缓存里反而留下旧值。用 @CacheEvict 先更新数据库、再删缓存，下次读时按 Cache Aside 回源重建，顺序错乱的影响只剩短暂不一致，配合 TTL 兜底更稳妥。@CachePut 只适合单写、低并发的配置型数据。
3. 因为 Spring Cache 靠 AOP 代理生效，同类内 this.getItem() 自调用不走代理，@Cacheable 会静默不生效、且不报错。这与 @Transactional 自调用是同一个病根，解法同样是注入自身代理或把方法拆到另一个类；排查时还要先确认已加 @EnableCaching、方法非私有、类已被 Spring 管理。
4. 只到同一 JVM 内：sync = true 保证同一 JVM 内同 key 只有一个线程回源，其他线程等待。跨实例无效，多实例下热点 key 过期瞬间仍会并发打库，集群防击穿要靠分布式锁或逻辑过期，见 Redis 篇。
5. 三处：序列化、TTL、空值策略。默认 JDK 序列化在 Redis 里是二进制不可读、类全名变更后反序列化直接失败，应换成 GenericJackson2JsonRedisSerializer 存 JSON 并带类型信息；无 TTL 等于缓存永不过期、改了永远读旧值，且各缓存要分级错峰、避免一起过期形成雪崩；空值策略要按缓存分别定，disableCachingNullValues 与 unless = "#result == null" 会放弃防穿透，而缓存空值则要配短 TTL（如 1 分钟）来防穿透。

6. `refreshAfterWrite` 只对 `LoadingCache` 生效：到期后条目变「可刷新」而非过期，首次访问到旧条目时异步调 `CacheLoader.reload(K, V)` 刷新并立即返回旧值，刷新异常被记录后吞掉，旧值继续留着；`expireAfterWrite` 则是到期即过期，下次访问重新回源。Spring 默认的 `CaffeineCacheManager`（只配 spec、没配 loader）建的是普通 `Cache`，不是 `LoadingCache`——只写 spec 不配 `CacheLoader`，刷新永远不会发生；要生效需显式调 `CaffeineCacheManager.setCacheLoader(...)`。首次加载仍是同步的：缓存里没有该 key 时，`@Cacheable` 方法照常同步执行并回写，`refreshAfterWrite` 只管「已有旧条目的后续刷新」，管不着第一次。

## 10. 关联知识

- [Spring Boot](开发语言/Java/Spring/Spring-Boot.md)：自动配置与 starter，缓存与 Redis 的装配入口。
- [Redis](数据库/Redis/Redis.md)：底层存储本身的数据结构与使用。
- [Redis 缓存问题与分布式锁](数据库/Redis/Redis-缓存问题与分布式锁.md)：穿透/击穿/雪崩与一致性的系统级对策，与本篇的注解落法对应。
- [Spring 事务](开发语言/Java/Spring/Spring-事务.md)：代理自调用与切面顺序的同源问题。
