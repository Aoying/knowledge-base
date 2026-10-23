# Java 知识点地图

> Java 位于 `开发语言/Java/`，按知识点拆分：一篇一个知识点，`Java.md` 为语言基础与总索引。

## 知识点清单

- 语言基础：[[Java]]（含基础知识补强、排查三板斧、疑难杂症）
- 框架实战：[[Spring-Boot]]、[[MyBatis-Plus]]、[[Bean-Validation]]
- Spring 深度：[[Spring-事务]]、[[Spring-IOC与AOP]]
- 框架补件：[[Spring-Cache]]、[[Quartz定时任务]]、[[Java-IO与NIO]]
- JVM 排错：[[JVM-故障排查]]（与 [[JVM-GC调优]] 配套）
- 数据访问：[[Druid]]、[[多数据源]]
- 语言进阶：[[Async-异步]]、[[Stream与Optional]]、[[虚拟线程]]
- JVM：[[JVM-GC调优]]、[[JVM-内存模型]]、[[JVM-类加载]]、[[JVM-字节码与JIT]]、[[JVM-故障排查]]
- 集合：[[HashMap]]、[[ConcurrentHashMap]]、[[ArrayList与扩容]]、[[LinkedHashMap与LRU]]、[[TreeMap与红黑树]]、[[Queue与Deque]]、[[CopyOnWriteArrayList]]（`开发语言/Java/集合/`）
- 并发：[[Java-并发编程]]（汇总，位于 `开发语言/Java/并发/`）；子篇：[[线程池]]、[[synchronized与锁]]、[[volatile与JMM]]、[[CAS与原子类]]、[[AQS]]、[[ThreadLocal]]
- 设计：[[设计模式]]（汇总；26 篇模式独立文档在 `开发语言/Java/设计模式/` 子目录）
- 工作流：[[Flowable]]（文件在工程实践，Java 体系内使用）
- 项目案例：[[神农-e贷-法人版]]、[[神农-e贷-审批流]]、[[督办系统]]

## 学习路径

```text
[[Java]]
  → [[Spring-Boot]] → [[MyBatis-Plus]] / [[Bean-Validation]]
  → [[Stream与Optional]] → [[Async-异步]] → [[虚拟线程]]
  → [[Java-并发编程]]
  → [[设计模式]]
  → [[Flowable]]（案例：[[神农-e贷-审批流]]、[[督办系统]]）
```

## 前置依赖

- [[数据结构与算法]]：集合与并发容器的底层结构
- [[操作系统-进程线程]]：线程、进程、调度基础
- [[计算机网络]]：分布式与 RPC 的前置

## 原理与实战关联

- 并发原理 → [[Java-并发编程]]；实战案例 → [[神农-e贷-审批流]] 中的多实例、并行网关处理
- 工作流原理 → [[Flowable]]；版本与兼容 → 各篇文末「版本差异」

## 知识缺口

- ~~JVM：JIT、字节码尚无独立篇~~（2026-10-23 已补 [[JVM-字节码与JIT]]，缺口移除）
