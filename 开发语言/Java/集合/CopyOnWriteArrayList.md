---
知识点: CopyOnWriteArrayList
层级: 进阶
适用版本: JDK 8+
更新日期: 2026-10-23
关联:
  - ArrayList与扩容
  - ConcurrentHashMap
  - Queue与Deque
标签:
  - Java
  - 集合
  - 并发
---

# CopyOnWriteArrayList

## 1. 核心概念

- `CopyOnWriteArrayList` 是 [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md) 的线程安全对应物，位于 `java.util.concurrent` 包。
- 核心机制写时复制：任何修改（add/set/remove）都先把底层数组整表拷贝一份，在副本上改，改完再把引用切到新数组。读操作完全无锁，直接读当前数组。
- 适用画像非常窄：**读极多、写极少、数据量小**。偏离这个画像就是性能灾难，选它之前先对照第 5 节。
- 允许 null 元素；迭代器是快照迭代器（见第 3 节），弱一致，不抛 `ConcurrentModificationException`。

## 2. 基础知识

```java
List<String> list = new CopyOnWriteArrayList<>();
list.add("a");
list.addIfAbsent("a");     // 独有方法：不存在才添加，检查+添加是原子的
list.addAllAbsent(List.of("a", "b")); // 批量版，只加不存在的
list.get(0);               // 无锁读
```

- 复合操作注意：单个方法是线程安全的，但 `if (!list.contains(x)) list.add(x);` 这种检查再添加仍非原子，要用 `addIfAbsent`，和 [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md) 里「单个方法原子 ≠ 复合操作原子」是同一条纪律。
- 配套类 `CopyOnWriteArraySet` 底层就是它（元素去重靠线性扫描的 contains 判断），同样只适合小数据量。

## 3. 进阶知识

### 快照迭代器

- 迭代器创建时记住当时的数组引用，之后集合怎么改，迭代看到的都是那一份快照：不反映新增，也看不到删除。
- 因此迭代器不支持 `remove`（调了抛 `UnsupportedOperationException`）——它操作的是过期快照，删谁都没有意义。
- 好处：迭代期间零锁、零干扰，长遍历不会拖住写线程。代价：读到的可能是旧数据，业务必须能容忍。

## 4. 原理剖析

```java
// add 简化流程
lock.lock();                        // 写锁：同一时刻只有一个写者（锁实现随版本有变化，语义不变）
try {
    Object[] old = getArray();
    Object[] copy = Arrays.copyOf(old, old.length + 1);
    copy[old.length] = e;
    setArray(copy);                 // volatile 引用一次性切换
} finally { lock.unlock(); }
```

- 数组引用字段是 `volatile` 的：切换后其他线程的读立刻看到新数组；单个元素在数组内的位置不变，读不需要任何同步。
- 每次写都是 O(n) 整表拷贝 + 新数组分配：写 n 次就是 O(n²) 的拷贝量，还持续制造垃圾数组给 GC。
- 与 [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md) 对照：CHM 是分桶细化锁、读写并行；COW 是写全量复制换读零成本，两个极端，不要混用场景。

## 5. 实战应用

适合的典型场景：

- 事件监听器列表：注册/注销偶尔发生，事件触发时高频遍历调用。
- 读多写少的配置/白名单：启动加载、偶尔热更新，请求路径上反复查。
- 路由表、策略表这类小规模、变更低频的数据。

不适合：

- 写频繁（日志收集、任务列表）：拷贝成本直接爆炸，用 [Queue与Deque](开发语言/Java/集合/Queue与Deque.md) 的阻塞队列或 CHM。
- 数据量大：哪怕写得少，单次拷贝大数组也会造成毛刺；考虑不可变快照 + volatile 引用手动实现，或读写锁保护普通 ArrayList。

## 6. 疑难杂症

### 场景 1：换上 COW 后吞吐反而暴跌

- 原因：写比例被低估。先统计真实读写比，写超过个位数百分比就该回滚到 `Collections.synchronizedList`、读写锁方案或 CHM 家族。
- 排查：看 GC 日志里 `Object[]` 分配是否异常增多，这是 COW 写拷贝的典型指纹。

### 场景 2：遍历时拿不到刚 add 的元素

- 不是 bug：迭代器是创建时的快照。要求读到最新数据的遍历，每次都新建迭代器或用下标 get 循环（get 读的永远是当前数组）。

## 7. 常见误区

- 误区：线程安全就等于万能。它只解决并发正确性，写成本和数据新鲜度是另付的价钱。
- 误区：迭代器弱一致 = 随便改不用管。遍历期间写线程照常全量拷贝，频繁写+长遍历叠加时拷贝开销翻倍。
- 误区：用它装大集合再偶尔改一次就行。单次拷贝的延迟毛刺也可能打到 P99，先算数据量。

## 8. 性能优化

- 批量写合并：能一次 `addAll` 就不要逐个 add，每次 add 都是一次全量拷贝，这是最大的优化点。
- 容量无法预分配（无 initialCapacity 的数组复用机制，构造传容量只影响初始数组），数据量大从一开始就别选它。

## 9. 版本差异

- 机制自 JDK 5 引入后不变；内部写锁的实现细节在后续版本有调整（锁字段形式不同），不影响语义与选型结论。

## 10. 面试与自测

- 写时复制的完整流程是什么？读为什么不用加锁？
- 它的迭代器和 ArrayList 的 fail-fast 迭代器有什么本质区别？
- 什么场景适合 COW？什么场景用它必翻车？
- `addIfAbsent` 解决了什么复合操作问题？
- 它与 ConcurrentHashMap 的并发思路有什么不同？分别适合什么读写画像？
- 自测：一个线程循环 add，另一个线程用迭代器遍历，验证遍历内容恒为某时刻快照且永不抛异常。

## 11. 关联知识

- [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md)：单线程原型；fail-fast 与快照迭代器的对照看这两篇。
- [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)：高并发通用解，与 COW 的窄场景形成对比。
- [Queue与Deque](开发语言/Java/集合/Queue与Deque.md)：写多读多的任务型数据应走阻塞队列，不是 COW。
