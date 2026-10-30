---
知识点: ThreadLocal
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - Java-并发编程
  - 线程池
标签:
  - Java
  - 并发
  - 实践
  - 排错
---

# ThreadLocal

## 1. 核心概念

- 定位：**以线程为作用域的变量副本**——每个线程通过同一个 `ThreadLocal` 对象读写到的，是只属于自己的一份值。它换的是思路：不解决共享冲突，而是根本不共享。
- 典型用途：请求上下文（用户 id、trace id、租户号）跨多层方法传递而不用层层加参数；给线程不安全的对象（如 `SimpleDateFormat`）做线程隔离。
- 它不是锁的替代品：该共享的状态还得共享，ThreadLocal 只适合「这份数据本来就是每个线程各用各的」场景。

## 2. 基础用法

```java
public class UserContext {
    private static final ThreadLocal<Long> CURRENT_USER = new ThreadLocal<>();

    public static void set(Long userId) { CURRENT_USER.set(userId); }
    public static Long get()             { return CURRENT_USER.get(); }
    public static void clear()           { CURRENT_USER.remove(); }   // 必须有人调它，见第 4 节
}

// 拦截器/过滤器入口设置，请求结束（finally）清理 —— Web 项目的标准姿势
UserContext.set(loginUserId);
try {
    chain.doFilter(req, res);
} finally {
    UserContext.clear();
}
```

- `withInitial`（JDK 8）：`ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"))` 免去判空初始化。
- `InheritableThreadLocal`：新建子线程时把父线程的值拷贝一份过去；注意只是创建那一刻的浅拷贝，且与线程池完全不兼容（见第 4 节）。

## 3. 原理剖析

- 数据不在 `ThreadLocal` 对象里，而是挂在**每个 `Thread` 对象自己的 `threadLocals` 字段**（`ThreadLocalMap`）上：key 是 ThreadLocal 实例（弱引用），value 是实际值。`get/set` 都先取当前线程的 map 再查。
- `ThreadLocalMap` 是开放地址法哈希表（线性探测），没有链表；哈希冲突靠往后挪格解决。所以 ThreadLocal 数量很多时它的表现不如 HashMap，别往一个线程里塞成百上千个。
- key 用弱引用的原因：ThreadLocal 对象本身没人引用时可以被 GC，不至于永远占着 map 的槽位；但 value 不是弱引用——这正是泄漏的来源。

## 4. 内存泄漏与线程池的坑（本篇重点）

- **泄漏链条**：线程（池里常驻，不死）→ ThreadLocalMap → Entry 的 key 已被回收变 null、value 还在 → 这条 Entry 永远取不到、删不掉，value 挂着对象无法回收。set/get 时有启发式清理能顺手扫掉一部分过期 Entry，但不能指望它兜底。
- **更现实的事故是串数据而不是 OOM**：[线程池](开发语言/Java/并发/线程池.md)复用线程，上一个请求 set 的用户信息没 `remove()`，下一个请求复用同一线程且入口漏 set 时，就会读到上一个用户的值——这是权限与隐私级事故，比泄漏更常见。
- 铁律：**凡 set 必有 finally remove**，收口放在过滤器/拦截器/切面统一做，不靠业务代码自觉。

```java
// 并发任务里的安全姿势：自己提交的任务，自己清理
pool.execute(() -> {
    try {
        UserContext.set(uid);
        doBusiness();
    } finally {
        UserContext.remove();
    }
});
```

## 5. 跨线程传递：TransmittableThreadLocal

- 问题：`InheritableThreadLocal` 只在**新建线程**时继承，而线程池的线程是复用的、早建好了，父线程提交任务时的上下文根本传不进池线程；异步一跳，trace id、用户信息全丢。
- 解法（阿里 TTL）：用 `TransmittableThreadLocal` 存值，再用 `TtlExecutors.getTtlExecutorService(pool)` 包装线程池——提交任务时捕获当前线程的全部 TTL 值、执行前回放到工作线程、执行后恢复，把池线程当一次性线程用。

```java
static final TransmittableThreadLocal<String> TRACE = new TransmittableThreadLocal<>();
ExecutorService ttlPool = TtlExecutors.getTtlExecutorService(rawPool);
TRACE.set("trace-123");
ttlPool.execute(() -> System.out.println(TRACE.get()));   // 池线程里也能拿到 trace-123
```

- 用了 TTL 也仍然要 `remove()`：它解决的是「传得进去」，不解决「残留串扰」，执行后恢复机制反而更依赖入口值的正确设置。
- 框架侧对应物：Spring 的 `TaskDecorator`、链路追踪组件（MDC 透传）本质都在解同一个问题；自研时优先 TTL，接入现成框架时先查框架是否已内置传递，别叠加两套。

## 6. 实战应用

- **项目落点**：法人版这类多租户/多机构系统，机构号、柜员号、trace id 放在 ThreadLocal 里贯穿 Controller→Service→DAO，审计日志与 SQL 拦截器直接从上下文取，避免方法签名全线加参数；收口在网关过滤器统一 set/remove。
- `SimpleDateFormat` 包 ThreadLocal 是老经验；新代码更推荐不可变的 `DateTimeFormatter`（天然线程安全，根本不用包）。
- 排查口诀：出现「偶发串号、重启就好、压测才复现」，先查 ThreadLocal 清理点；出现「老年代缓慢上涨且堆里大量业务对象被 ThreadLocalMap 引用」，就是泄漏，`jmap` 堆转储顺引用链一查便知。

## 7. 常见误区

- 以为 ThreadLocal 能「让变量线程安全」：它只是隔离，不是同步；把共享累加器放进去，各线程各加各的，结果照样错。
- 只在正常路径 remove：异常路径漏清等于没清，remove 必须进 finally。
- 在异步/池场景指望 InheritableThreadLocal：继承只发生在 new Thread 那一刻，池线程不新建，必丢。
- 值里放大对象：value 的生命周期等于线程（池线程 ≈ 进程），大对象、连接、流句柄都不该放进来。

## 小技巧与冷知识

【小技巧】没 set 过的 `ThreadLocal.get()` 返回 null，不抛异常：普通 `new ThreadLocal<>()` 第一次 get 拿到的是 null，业务代码里直接 `.toString()` 或拆箱就会 NPE。原文第 2 节的 `withInitial` 就是治这个的——它给每个线程一个初始值，get 永远非空；不确定调用方会不会先 set 时，优先用 withInitial，别在每个 get 点写判空。

【小技巧】`set(null)` 不等于 `remove()`：set(null) 只是把 Entry 的 value 写成 null，这条 Entry 还占着 map 里的槽位、还计入大小；`remove()` 才是真正把 Entry 从 map 里删掉。清理时永远调 remove，别用 set(null) 代替——后者留下的空 Entry 只有等启发式清理来扫，池线程里攒多了照样占地方。

【小技巧】日志的 MDC 也是 ThreadLocal，漏清会串日志：SLF4J 的 MDC（`MDC.put("traceId", ...)`）底层就是 ThreadLocal，拦截器里 put 了 traceId，请求结束必须在 finally 里 `MDC.clear()`。漏清的后果和原文第 4 节一样——池线程复用后，下一条请求的日志会带上上一条请求的 traceId，链路追踪直接串线；MDC 和业务 ThreadLocal 要放在同一个 finally 里清。

【小技巧】随机数别用 `ThreadLocal<Random>` 包，自己用 `ThreadLocalRandom.current()`：它是 JDK 专门给并发写的随机数生成器，不走 ThreadLocalMap，没有 Entry、不用 remove，也没有 `Math.random()` 的共享锁竞争。看到老代码里 `ThreadLocal.withInitial(Random::new)` 可以直接换成 `ThreadLocalRandom.current().nextInt(n)`，少一个泄漏点。

【小技巧】`InheritableThreadLocal` 的子线程值默认是浅拷贝，想深拷贝要重写 `childValue()`：父线程传一个可变对象（比如 Map、List）给子线程，子线程改了它，父线程的值也跟着变——因为拷贝的只是引用。值是可变对象且子线程会改时，重写 `childValue()` 返回一份深拷贝；只读场景用默认浅拷贝就行，拷贝成本和安全性要先想清楚。

【小技巧】`CompletableFuture` 的回调换了线程，ThreadLocal 跟不过去：`thenApply` 在哪个线程跑取决于前一个阶段的完成线程，`thenApplyAsync` 明确换线程池，两者都不会继承调用线程的 ThreadLocal 值。异步链路里要传 traceId、用户信息，要么在回调入口手动 set、出口手动 remove，要么用原文第 5 节的 TTL 包装任务；指望 ThreadLocal 自己跨过异步边界必丢。

【小技巧】ThreadLocal 字段一律写成 static final，别每个对象 new 一个：ThreadLocal 的身份就是它自己——每个实例在每个线程的 map 里都占一条 Entry。把 ThreadLocal 写成实例字段，每 new 一个业务对象就多一个 key，map 里攒一堆只用一次的 Entry；static final 一份，全线程共用同一个 key，Entry 只有一条，清理时也只要 remove 一次。

【小技巧】ThreadLocalMap 的表只长不缩：表满了触发 rehash 时容量翻倍，但之后哪怕大部分 Entry 被清掉，表的长度也不会缩回去。一个长期存活的池线程如果历史上塞过大量不同的 ThreadLocal 实例，它的 map 表会一直保持在峰值大小。控制 ThreadLocal 实例总数（复用 static 字段、少建临时实例），比事后指望 map 自己缩回去靠谱。

【小技巧】虚拟线程（JDK 21+）里慎用 ThreadLocal：每个虚拟线程也有自己的一份 ThreadLocalMap，虚拟线程动辄几十万个，每个都挂一份值，内存是按线程数乘出来的。虚拟线程场景下传上下文优先用 `ScopedValue`（JDK 21 预览，JDK 25 正式）：值绑定在作用域上、退出自动失效，不用 remove，也没有残留串扰。

【冷知识】每个 ThreadLocal 的哈希增量是 0x61c88647：JDK 用斐波那契散列给每个 ThreadLocal 实例分配 `threadLocalHashCode`，新实例的哈希在上一个基础上加 0x61c88647（黄金分割比的 2^32 倍取整）。这个增量让连续创建的 ThreadLocal 实例在 map 里的槽位均匀散开，配合原文第 3 节的线性探测，冲突时往后挪的格数最少。

【冷知识】ThreadLocalMap 初始容量 16，扩容阈值是容量的三分之二：表里 Entry 数超过 `容量 × 2/3` 就触发 rehash，容量翻倍。结合“表只长不缩”那条看，一个线程里 ThreadLocal 实例数长期超过 10 个左右，map 就会从 16 扩到 32——数量本身就是性能信号，塞成百上千个时线性探测的步数会明显变长。

【冷知识】`ScopedValue` 是 ThreadLocal 的官方继任者之一：JDK 21 以预览特性引入（JEP 446），JDK 25 正式落定（JEP 506），专为虚拟线程和结构化并发设计。它和 ThreadLocal 的根本区别是不可变——值只能在作用域内读、不能中途改，作用域退出自动失效，从机制上杜绝了“忘记 remove”和“跨请求串数据”这两类事故。

## 8. 面试与自测

- ThreadLocalMap 的 key 为什么是弱引用？value 为什么仍会泄漏？
- 为什么 InheritableThreadLocal 配合线程池失效？TTL 怎么补的？
- 你项目里 ThreadLocal 的清理收口在哪一层？异常路径覆盖了吗？

### 参考答案

1. key 用弱引用，是为了让 ThreadLocal 实例本身没人引用时可以被 GC，不至于永远占着 map 的槽位。但 value 是强引用：key 被回收变 null 后，这条 Entry 取不到也删不掉，value 挂着对象无法回收；线程池线程常驻不死，这条泄漏链就一直存在。set/get 的启发式清理只能顺手扫掉一部分过期 Entry，不能指望它兜底。
2. InheritableThreadLocal 只在新建线程那一刻把父线程的值浅拷贝一份；线程池的线程早建好了、长期复用，提交任务时父线程的值根本传不进池线程，所以失效。TTL 的补法：用 TransmittableThreadLocal 存值，再用 TtlExecutors 包装线程池——提交时捕获当前线程的全部 TTL 值，执行前回放到工作线程、执行后恢复。但 TTL 只解决“传得进去”，不解决残留串扰，仍要 remove。
3. 按正文推荐的收口：网关过滤器/拦截器/切面统一做——入口 set，请求结束在 finally 里 remove，不靠业务代码自觉；自己提交的并发任务，自己在 finally 里清理。异常路径必须覆盖：只在正常路径 remove 等于没清，remove 必须进 finally。（本题问的是本项目实际做法，正文只给了推荐收口，答案按正文写法作答，需自行对照项目现状。）

## 9. 关联知识

- 汇总与全景：[Java-并发编程](开发语言/Java/并发/Java-并发编程.md)
- 复用线程的来源：[线程池](开发语言/Java/并发/线程池.md)
- 异步编排中的上下文：[Async-异步](开发语言/Java/Async-异步.md)
