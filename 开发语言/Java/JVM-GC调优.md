---
知识点: JVM GC 调优
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-22
关联:
  - Java
  - Java-并发编程
标签:
  - Java
  - JVM
  - 性能优化
---

# JVM GC 调优速查

## JVM GC 调优速查

```bash
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar
```

- **Xms=Xmx**：别让堆动态伸缩，伸缩时的 Full GC 最伤。
- **G1** 是 8+ 默认，`MaxGCPauseMillis=200` 给目标值（不是保证值）。
- 看 GC 日志：`-Xlog:gc*:file=gc.log:time`，找"停顿>1s"和"Full GC 频繁"。
- 神农这种长跑服务：先保证堆够（`jstat -gc` 看 Old 区涨速），再调暂停目标，别反过来。

