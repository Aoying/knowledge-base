---
知识点: JVM 故障排查
层级: 进阶
适用版本: JDK 8 / JDK 11（项目 TongWeb + Spring Boot 2.7.3）
更新日期: 2026-10-23
关联:
  - JVM-GC调优
  - JVM-内存模型
  - 监控告警
标签:
  - Java
  - JVM
  - 排错
  - 性能优化
---

# JVM 故障排查

## 1. 核心概念

线上 JVM 故障主要是三类：**CPU 飙高**（死循环、频繁 GC、正则回溯、线程打架）、**内存溢出**（堆、元空间、直接内存、线程数）、**频繁 GC / 停顿长**（对象分配过快、老年代回收不掉、大对象）。排查的总原则：**先留现场，再下结论，最后才重启**——堆栈、堆快照、GC 日志一旦进程没了就再也拿不到。

排查工具分三层：操作系统（top/free）、JDK 自带（jstack/jmap/jstat/jinfo，零依赖、生产可用）、Arthas（在线诊断，能看方法级耗时与入参，适合预发与经审批的生产）。

## 2. 基础知识

### 命令清单

| 命令 | 用途 | 常用形式 |
|---|---|---|
| `top -Hp <pid>` | 找吃 CPU 的线程（拿十进制 TID） | `top -Hp 12345` |
| `jstack <pid>` | 线程快照：死锁、阻塞、热点栈 | `jstack -l 12345 > stack.txt` |
| `jmap -heap <pid>` | 各代内存占用概览 | JDK 11 已弱化，优先看 GC 日志/jstat |
| `jmap -dump` | 导出堆快照给 MAT 分析 | `jmap -dump:format=b,file=heap.hprof 12345` |
| `jstat -gcutil <pid> 1000` | 每秒看各区使用率与 GC 次数/耗时 | 判断是否频繁 Full GC |
| `jinfo -flags <pid>` | 运行时 JVM 参数确认 | 核对线上到底用了什么参数 |
| Arthas `thread -n 3` | 直接给出最忙的 3 条线程栈 | 替代手算 nid 的快捷路径 |
| Arthas `trace/watch` | 方法级耗时与入参观察 | `trace 包.类 方法 '#cost > 50'` |
| Arthas `dashboard` | 实时总览：线程、内存、GC | 初到现场第一眼 |

### GC 日志开启（先开着，出事才有据可查）

JDK 8（项目当前基线）：

```bash
-Xloggc:/data/logs/gc-%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=20M
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:ErrorFile=/data/logs/hs_err_%p.log
```

JDK 9+ 统一日志：`-Xlog:gc*:file=/data/logs/gc-%t.log:time,uptime:filecount=10,filesize=20M`（其余参数同上）。

## 3. 进阶知识

### GC 日志读法

一条 Young GC 日志看四件事：

1. **频率**：两次 GC 间隔越来越短 = 分配速率过高或堆太小。
2. **回收效果**：GC 后老年代占用是否回落；只涨不落 = 有对象持续晋升且回收不掉，怀疑泄漏。
3. **单次停顿**：看 `real` 时间；停顿长但 GC 时间（user）短，多是机器抢 CPU 或日志 IO。
4. **Full GC 触发原因**：日志里的 `Metadata GC Threshold`（元空间不足）、`System.gc()`（代码或框架显式调用，排查 RMI/框架配置）、`Allocation Failure`（老年代装不下晋升对象）指向完全不同的解法。

判断泄漏的金标准不是单次快照，而是**多次 Full GC 后老年代下限持续抬升**。结合 `jstat -gcutil` 看 O（老年代使用率）在 Full GC 后是否一次比一次高。

### OOM 类型对照

| 报错 | 含义 | 先查什么 |
|---|---|---|
| `Java heap space` | 堆装不下 | heap dump 看大对象与引用链 |
| `GC overhead limit exceeded` | GC 拼命跑但回收不到 2% | 同上，基本是泄漏临界点 |
| `Metaspace` | 类元数据区满 | 类加载失控：动态代理/字节码生成、类加载器泄漏 |
| `Direct buffer memory` | 堆外直接内存满 | NIO/Netty 使用，`-XX:MaxDirectMemorySize` 与释放 |
| `unable to create new native thread` | 线程数到上限 | 线程池无界、每请求建线程；看 `ulimit -u` 与线程数 |
| `StackOverflowError` | 栈溢出 | 递归无终止、循环调用（不是内存问题） |

## 4. 实战应用

### CPU 飙高定位流程

```bash
top                              # 1. 找到吃 CPU 的 Java 进程 PID
top -Hp <pid>                    # 2. 找到最忙线程 TID（十进制）
printf '%x\n' <tid>              # 3. TID 转十六进制 = 线程栈里的 nid
jstack <pid> | grep -A 30 'nid=0x<hex>'   # 4. 看这条线程在跑什么代码
```

连续抓 3 次 jstack（间隔几秒）：栈总停在同一业务方法 = 死循环或热点计算；总在 GC 线程 = 问题在 GC 而不在业务（转去看堆与 GC 日志）；大量线程 BLOCKED 在同一把锁 = 锁竞争，转查持锁线程。

Arthas 路径更短：`thread -n 3` 一步给出最忙线程及栈；再 `trace` 到可疑方法看内部耗时分布。

### 内存溢出定位流程

1. 先看 OOM 报错类型（对照上表）缩小范围，别一上来就 dump。
2. 堆问题：用自动留下的 `HeapDumpOnOutOfMemoryError` 文件，或手动 `jmap -dump`（大堆 dump 会长时间停顿，生产先报备）。
3. MAT 打开 dump：看 Histogram 第一名是谁、Dominator Tree 的 Retained Heap、Leak Suspects 报告；沿 GC Roots 引用链确认「谁在持有、本该何时释放」。
4. 常见真凶：静态集合只加不删、缓存无上限无过期、ThreadLocal 用完不 remove、监听器/回调注册后不注销、大结果集一次全量查回内存（分页缺失）。

### 频繁 GC / 接口毛刺定位流程

1. `jstat -gcutil <pid> 1000 20` 看 YGC/FGC 频率与耗时增量。
2. GC 日志确认停顿与接口毛刺时间点是否对齐（与 [监控告警](工程实践/监控告警.md) 的 GC 指标大盘互相印证）。
3. 分清三种成因：分配太快（短命对象洪峰，如大 JSON 拼装、日志狂打字符串）、老年代泄漏式增长（转内存流程）、单次大对象（导出、整表查询、附件全量进内存）。
4. 解法分别对应：优化分配热点、修泄漏、导出改流式/分页；调大堆只是给前两类续命，不解决问题本身。

### 排查纪律

- 先保存现场（jstack 三连、GC 日志、dump 路径记清），再允许重启恢复服务；重启前没留的东西事后无法复盘。
- 生产执行 `jmap -dump`、Arthas `redefine` 等高风险动作要走审批并避开业务高峰；dump 文件含业务数据与敏感信息，按敏感文件管理、分析完销毁。
- 结论要能落到代码行或配置项：「机器不行」不是结论。

## 5. 常见误区

- 出事第一反应重启：服务活了，证据没了，同类故障必然复发且每次都从零查。
- 看到 Full GC 多就调大堆：泄漏型增长调多大都会满，只是把爆炸时间推后。
- 单次 jstack 下结论：一次快照只能证明「那一刻」，热点判断靠多次采样一致性。
- 用 `System.gc()` 在代码里「帮忙回收」：触发 Full GC 的停顿远比省下的内存贵，框架（如 RMI）默认周期调用也要显式关掉或配 `-XX:+DisableExplicitGC` 前先想清副作用。

## 6. 面试与自测

1. CPU 100% 的完整定位链路（从 top 到代码行）说一遍。
2. 怎么判断是内存泄漏而不是内存不足？看哪几个指标、什么趋势？
3. `Metaspace` OOM 与堆 OOM 的成因和查法有什么不同？
4. GC 日志里 Full GC 只涨不落说明什么？下一步做什么？
5. 为什么生产环境要预先开启 GC 日志和 HeapDumpOnOutOfMemoryError？

## 7. 关联知识

- 前置：[JVM 内存模型](开发语言/Java/JVM-内存模型.md)、[JVM GC 调优](开发语言/Java/JVM-GC调优.md)
- 相关：[监控告警](工程实践/监控告警.md)（GC 与饱和度指标、告警阈值）
- 延伸：MAT 深入分析、火焰图（async-profiler）、JFR 持续记录
