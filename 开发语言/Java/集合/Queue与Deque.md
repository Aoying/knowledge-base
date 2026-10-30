---
知识点: Queue与Deque
层级: 基础
适用版本: JDK 8+
更新日期: 2026-10-31
关联:
  - ConcurrentHashMap
  - CopyOnWriteArrayList
  - ArrayList与扩容
标签:
  - Java
  - 集合
  - 并发
---

# Queue与Deque

## 1. 核心概念

- `Queue` 是单端队列：一端进（队尾）、一端出（队头），先进先出。每组操作都有两个版本，这是使用 Queue 的第一课：

| 语义 | 失败抛异常 | 失败返回特殊值 |
|---|---|---|
| 入队 | `add` | `offer`（成功 true，满则 false） |
| 出队 | `remove` | `poll`（空则 null） |
| 看队头 | `element` | `peek`（空则 null） |

- `Deque` 是双端队列，继承 Queue，两头都能进出（方法名带 `First`/`Last`）。只操作一端是栈，两端分工就是普通队列。
- 选型总览：普通队列/栈 → `ArrayDeque`；优先级 → `PriorityQueue`；并发阻塞 → `BlockingQueue` 家族（见第 3 节）。

## 2. 基础知识

```java
Deque<String> stack = new ArrayDeque<>();
stack.push("a"); stack.push("b");
stack.pop();                 // "b"，栈语义

Queue<String> q = new ArrayDeque<>();
q.offer("x");
q.peek();                    // "x"，不移除
q.poll();                    // "x"，移除
```

### ArrayDeque

- 循环数组实现：`head`、`tail` 两个下标在数组上环形移动，满时翻倍扩容。
- 不允许 `null` 元素：`poll`/`peek` 用 null 表示空队列，存 null 会产生歧义，与 [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md) 禁 null 是同一个理由。
- 做栈和队列都比 `LinkedList` 快（连续内存、无节点对象），也比老 `Stack`（继承 Vector、全方法 synchronized）干净。官方文档明确推荐用 Deque 替代 Stack。

### PriorityQueue

- 二叉堆实现（数组存储），默认最小堆：队头永远是最小元素，`offer`/`poll` O(log n)，`peek` O(1)；排序规则坑同 [TreeMap与红黑树](开发语言/Java/集合/TreeMap与红黑树.md)：比较为 0 即同优先级。
- 注意：只有队头有序，遍历顺序无意义；不允许 null；非线程安全，并发版是 `PriorityBlockingQueue`。

## 3. 进阶知识：BlockingQueue 家族

`BlockingQueue` 继承 Queue 并加阻塞语义：队列空时 `take` 阻塞等待，队列满时 `put` 阻塞等待——它就是生产者-消费者模型的标准件：

| 实现 | 结构 | 容量 | 特点 |
|---|---|---|---|
| `ArrayBlockingQueue` | 数组 + 一把锁 | 有界（构造时定） | 可选公平锁，背压首选 |
| `LinkedBlockingQueue` | 链表 + 两把锁（put/take 分离） | 默认近乎无界 | 吞吐高，不设界等于没背压 |
| `PriorityBlockingQueue` | 堆 + 一把锁 | 无界 | 按优先级取，put 永不阻塞 |
| `DelayQueue` | 堆 | 无界 | 元素实现 `Delayed`，到期才能取 |
| `SynchronousQueue` | 无存储 | 0 | put 必须等 take 直接交接 |
| `LinkedTransferQueue` | 链表 | 无界 | `transfer` 等消费者确认 |

### 与线程池的呼应

- `ThreadPoolExecutor` 的任务队列就是 `BlockingQueue<Runnable>`，预置组合直接决定行为：`newFixedThreadPool` 用无界 `LinkedBlockingQueue`（任务无限堆积，风险藏在队列里）；`newCachedThreadPool` 用 `SynchronousQueue`（不排队，风险转移到线程数）；`newScheduledThreadPool` 用 `DelayQueue` 的变体。
- 生产实践：手写线程池用有界 `ArrayBlockingQueue` + 明确拒绝策略，让过载显形，不要把问题推迟成 OOM。

## 4. 原理剖析

- `ArrayDeque` 出入队只动 head/tail 下标，不搬移元素，均摊 O(1)；扩容时才按顺序拷贝到新数组并重置 head=0。
- `PriorityQueue` 堆操作：入队放数组尾再上浮（siftUp），出队用尾元素补根再下沉（siftDown)，比较 O(log n)。
- `ArrayBlockingQueue` 用一把 `ReentrantLock` + `notEmpty`/`notFull` 两个 Condition 协调阻塞唤醒；`LinkedBlockingQueue` 把 put/take 拆成两把锁各管一端，降低互斥。

## 5. 实战应用

- 任务缓冲、请求排队、层序遍历（BFS）都用 Queue；不要拿 [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md) 的 `remove(0)` 模拟队列，每次出队搬移全体元素，O(n)。
- 限时等待用 `offer(e, timeout, unit)` / `poll(timeout, unit)`，超时返回特殊值，方便做退出与降级。

## 6. 疑难杂症

### 场景 1：线程池任务越积越多最终 OOM

- 原因：无界队列下线程数永远不会涨过核心数，拒绝策略永远不触发，任务只进不出地堆积。
- 修复：换有界队列，让拒绝策略在过载时生效，并监控队列长度。

### 场景 2：poll 拿到 null 不知是空还是元素就是 null

- 标准队列实现大多禁 null 正是为此；真出现 null，多半是有人塞进了允许 null 的实现（如 LinkedList），统一换 ArrayDeque 或 BlockingQueue。

## 7. 常见误区

- 误区：`add` 和 `offer` 可以混用。队列满时一个抛异常一个返回 false，控制流完全不同。
- 误区：PriorityQueue 遍历出来是有序的。只有依次 poll 才有序。
- 误区：LinkedBlockingQueue 是有界的。默认容量 `Integer.MAX_VALUE`，要背压必须构造时传容量。

## 8. 性能优化

- 非阻塞场景 ArrayDeque 是最快的通用队列，不要默认用 LinkedList。
- 阻塞队列容量按「可承受的最大排队时长 × 消费速率」估，不要凭感觉给个大数；公平锁明显降吞吐，非必要不开。

## 9. 版本差异

- JDK 7 新增 `LinkedTransferQueue` 与 `ArrayDeque` 的双端 API 定型；核心家族自 JDK 5 引入后无结构性变化。

## 小技巧与冷知识

【小技巧】有界队列的入队优先用 offer 而不是 add：add 在队列满时抛 IllegalStateException，调用方必须 try-catch 才能走降级分支；offer 满时只返回 false，一个 if 就能接住“放不下就丢弃/记指标/走降级”。异常是给“不该发生的事”准备的，队列满在高并发下是常态，用返回值处理常态、用异常处理异常，别把控制流写进 catch 里。

【小技巧】队列空到底是“正常空闲”还是“出 bug 了”，用 poll 还是 remove 来区分：poll 空返回 null，适合“空了就等一会儿再来”的消费循环；remove 空抛 NoSuchElementException，适合“这里一定该有元素、没有就是上游出问题”的场景。反过来用就错位——消费循环里用 remove 会把正常空闲变成异常日志刷屏，断言“必须有”时用 poll 会把 bug 静默成 null 往下传。

【小技巧】批量消费别手写 while + poll，用 drainTo：BlockingQueue 的 drainTo(Collection) 一次把队里现有元素批量取走，内部只加一次锁；手写循环每次 poll 都要单独加锁、单独判空。消费端想“攒一批再处理”（批量写库、批量发请求）时，drainTo 是现成的批量出口；带上限的 drainTo(c, maxElements) 还能控制一批最多取多少，防止一批过大。

【小技巧】消费者退出循环别用 take()，用带超时的 poll：take() 在队列空时会一直阻塞，停机信号来了线程还在里面睡着，只能靠 interrupt 叫醒；poll(1, TimeUnit.SECONDS) 每次超时返回 null，正好顺手检查一次“是否收到停机标志”。需要立即退出的消费线程，退出检查要写进循环里，而不是寄希望于外部 interrupt 一定能及时送达。

【小技巧】生产者线程别在请求处理线程上直接调 put()：有界队列满时 put() 会阻塞直到有空位，请求线程被挂在队列上，超时、熔断都救不了它。生产者入队用 offer(e, timeout, unit)，超时返回 false 后直接走降级（返回“系统繁忙”、记指标），别让一次入队把调用线程无限期钉死；阻塞式入队只适合专门的生产者线程。

【小技巧】ArrayDeque 当栈用时，push/pop 和 addFirst/removeFirst 是同一对方法：push 就是 addFirst，pop 就是 removeFirst，空栈 pop 抛 NoSuchElementException；想“空了就返回 null”用 pollFirst。老 Stack 的 push/pop 语义被 Deque 原样继承了，换实现时方法名不用改，只改声明类型；但别把“栈顶”理解成数组末尾——ArrayDeque 的栈顶在 head 一端，出入都在同一端。

【小技巧】PriorityQueue 里优先级相同的元素，出队顺序不保证是先进先出：堆只保证队头最小，两个比较结果为 0 的元素谁先出来看堆内部排列。业务上“同优先级按到达顺序处理”时，比较器里要加一个单调递增的序列号做第二排序键（先比优先级、再比序号），只靠堆本身的排列顺序，顺序会在扩容、删除后悄悄变化。

【小技巧】DelayQueue 的元素到期时间要在构造时就定死，别在 getDelay 里现算：Delayed 的 getDelay 返回的是“距离到期还剩多久”，实现里通常用到期时刻减当前时间；到期时刻必须在元素创建时算好存起来。如果每次 getDelay 都拿“创建时间 + 延迟”现算，元素在队列里被反复比较时时间基准会漂移，到期判断就不再稳定；compareTo 也要和 getDelay 用同一套时间基准，否则堆排序和到期判断会打架。

【小技巧】SynchronousQueue 的 offer 几乎永远返回 false，别把它当缓冲用：它内部没有存储位置，offer 只有在正好有另一个线程在等 take 时才成功，其余时间直接返回 false。把它当普通队列 offer 入队，数据会悄悄丢失；它的正确用法是“交接”——生产者 transfer/put 等消费者来接，消费者 take 等生产者来给，线程池用它就是要“不排队、直接把任务交给空闲线程”。

【冷知识】ArrayDeque 的容量永远是 2 的幂：它内部数组长度只取 16、32、64 这样 2 的幂，head、tail 下标移动时用位运算 `& (elements.length - 1)` 代替取模，环形移动一次只要一条与指令。扩容翻倍后容量仍是 2 的幂，所以“满了就翻倍”不只是容量策略，也是为了让这个位运算一直成立；初始容量传 10 会被向上取到 16，传多少都不影响这个规矩。

【冷知识】Queue 和 BlockingQueue 是 Java 5（JSR 166，Doug Lea 主导）引入的，Deque 是 Java 6 才加进来的：Java 5 之前集合框架里只有 List、Set、Map，队列语义靠 LinkedList 硬凑；JSR 166 把并发工具整体带进 JDK，Queue、BlockingQueue、ConcurrentHashMap 是同一批货。Deque 接口与 ArrayDeque 在 Java 6 补齐，LinkedList 也就是在那时才正式实现 Deque——队列在 Java 里是“后来补上的一层抽象”，不是一开始就有的。

【冷知识】Deque 这个词读作“deck”：它是 double-ended queue 的缩写，官方发音按 deck 来读，不是逐字母念 D-E-Q。PriorityQueue、DelayQueue 这些名字都直白，只有 Deque 的读法容易念错；开会、面试时读“deck”就对了，读“迪克”会被当成没用过它。

【冷知识】DelayQueue 内部就是一台 PriorityQueue：它底层用一个 PriorityQueue 按到期时间排队，take 时只看堆顶元素是否到期，没到期就等着（用 Condition 精确等到最近那个元素的到期时刻）。所以 DelayQueue 的元素必须能比较大小——Delayed 接口继承了 Comparable，compareTo 排的是到期顺序；理解成“按时间排序的优先队列”就全通了。

## 10. 面试与自测

- Queue 三组方法（add/offer、remove/poll、element/peek）区别是什么？
- ArrayDeque 为什么不允许 null？为什么推荐它替代 Stack 和 LinkedList？
- ArrayBlockingQueue 与 LinkedBlockingQueue 的锁和容量区别？FixedThreadPool 的 OOM 风险从哪来？
- PriorityQueue 的底层是什么？为什么遍历无序？
- 自测：用 ArrayBlockingQueue 写生产者-消费者，容量设 2，观察 put/take 的阻塞点。

### 参考答案

1. 三组方法区别只在失败语义：入队 add 满抛异常、offer 满返回 false；出队 remove 空抛 NoSuchElementException、poll 空返回 null；看队头 element 空抛异常、peek 空返回 null。阻塞队列另有 put/take（阻塞等待）与带超时的 offer/poll。
2. ArrayDeque 不允许 null：poll/peek 用 null 表示空队列，存 null 会产生“空还是元素就是 null”的歧义，与 ConcurrentHashMap 禁 null 同一个理由。推荐替代 Stack：Stack 继承 Vector、全方法 synchronized 且是遗留类，Deque 更干净；替代 LinkedList：循环数组连续内存、无节点对象，出入队只动 head/tail 下标，均摊 O(1)，更快。
3. 锁与容量：ArrayBlockingQueue 是数组 + 一把 ReentrantLock + notEmpty/notFull 两个 Condition，有界（构造时定），可选公平锁；LinkedBlockingQueue 是链表 + put/take 两把锁分离，默认容量 Integer.MAX_VALUE、近乎无界，吞吐高但不设界等于没背压。FixedThreadPool 的 OOM 风险：它用无界 LinkedBlockingQueue，任务无限堆积、线程数永远涨不过核心数、拒绝策略永远不触发，风险藏在队列里，最终 OOM。
4. PriorityQueue 底层是二叉堆（数组存储），默认最小堆，队头永远是最小元素；offer/poll O(log n)（上浮/下沉），peek O(1)。遍历无序：只有队头有序，堆数组本身不按全序排列；只有依次 poll 才是从小到大的顺序；不允许 null，非线程安全，并发版是 PriorityBlockingQueue。
5. 自测思路：ArrayBlockingQueue 容量 2，生产者连续 put 3 个、消费者 take。验证要点：前 2 个 put 成功，第 3 个 put 在队列满时阻塞，直到消费者 take 腾出位置才继续；队列空时 take 阻塞，直到生产者 put 才返回；用带超时 offer/poll 还能观察超时返回特殊值。

## 11. 关联知识

- [ArrayList与扩容](开发语言/Java/集合/ArrayList与扩容.md)：数组结构与扩容思路，ArrayDeque 的循环数组是它的变体。
- [CopyOnWriteArrayList](开发语言/Java/集合/CopyOnWriteArrayList.md)：另一类并发容器，读无锁、写拷贝，与阻塞队列解决的问题不同。
- [ConcurrentHashMap](开发语言/Java/集合/ConcurrentHashMap.md)：同属 `java.util.concurrent`，禁 null 的理由一致；线程池与队列的搭配需结合 Java-并发编程中的线程池章节一起看。
