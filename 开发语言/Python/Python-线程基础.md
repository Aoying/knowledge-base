---
知识点: Python 线程基础
层级: 基础
适用版本: Python 3.10+
更新日期: 2026-10-25
关联:
  - Python
  - Python-多进程与GIL
  - asyncio
标签:
  - Python
  - 并发
  - 实践
---

# Python 线程基础

## 1. 核心概念：线程是什么

线程是进程内的执行单元，同一进程的线程共享内存：全局变量、堆上的对象互相可见，改起来方便，竞态也随之而来。线程适合“等外部”的活——调接口、等数据库、读写文件、sleep；不适合纯 Python 的算活，那是 GIL 的地盘，边界见第 6 节。

```python
import threading

def fetch(url: str) -> None:
    ...  # 阻塞 IO：发请求、等返回

t = threading.Thread(target=fetch, args=("https://example.com",))
t.start()  # 只负责启动，不等它跑完
t.join()   # 等它跑完；不 join，主线程跑完就直接往下走了
```

- **start() 与 join()**：`start` 是“开始跑”，`join` 是“等跑完”。主线程结束时，未 join 的非守护线程会拖住进程退出，直到它们跑完。
- **守护线程（daemon）**：`Thread(..., daemon=True)`，主线程退出时会被强制结束，不管手头活干没干完。只适合“死了也无妨”的后台活（心跳、日志刷盘），落盘、提交这类不能丢的活绝不能放守护线程。
- **线程不是免费的**：每个线程占一份栈内存（默认约 8MB 虚拟内存），几千个线程就不是线程池该管的量级了，那是 [asyncio 并发模型](开发语言/Python/asyncio.md) 的场景。

## 2. 同步原语：Lock、RLock、Event

### 2.1 Lock 与 RLock

```python
import threading

counter = 0
lock = threading.Lock()

def add() -> None:
    global counter
    with lock:  # with 块退出时自动释放，异常也不怕漏放
        counter += 1
```

- **Lock**：互斥锁，同一时刻只一个线程持有。`with lock` 是唯一推荐写法，裸 `acquire()` 配 `release()` 很容易在异常路径漏放。
- **RLock**：可重入锁，同一线程可多次 `acquire`，释放次数要配对。递归函数、或一个持锁方法调另一个也要持锁的方法时用它；普通场景用 Lock 就够，RLock 会掩盖“锁粒度过大”的设计问题。
- **死锁三件套**：两个线程互相等对方的锁、锁顺序不一致、`acquire()` 不带超时。防法：全局统一锁顺序、关键路径 `acquire(timeout=...)`、锁粒度尽量小、持锁期间不调外部回调。

### 2.2 Event：线程间的启停信号

```python
stop = threading.Event()

def worker() -> None:
    while not stop.is_set():
        ...  # 干活，循环里定期查信号

stop.set()  # 主线程通知：停吧
```

`Event` 是个布尔旗：`set()` 置位、`clear()` 复位、`wait(timeout)` 阻塞等信号。典型用法是优雅退出——工作线程循环里定期查 `is_set()`，主线程 `set()` 后再 `join()`，比粗暴杀线程干净得多。`wait(timeout)` 带超时还能顺带当“睡一会儿但可提前叫醒”用。

## 3. Queue：线程安全通信

线程之间不要用 `list` + `Lock` 自己拼队列，标准库 `queue.Queue` 本身就是线程安全的：

```python
import queue, threading

q: queue.Queue = queue.Queue(maxsize=100)

def producer() -> None:
    for item in jobs:
        q.put(item)  # 满了就阻塞等，不丢数据
    q.put(None)      # 哨兵：告诉消费者没活了

def consumer() -> None:
    while True:
        item = q.get()
        if item is None:
            q.task_done()
            break
        process(item)
        q.task_done()
```

- **生产者消费者标准结构**：生产者 `put`、消费者 `get`、干完调 `task_done()`、主线程 `q.join()` 等全部任务完成。`task_done()` 漏调一次，`join()` 就永远等下去。
- **哨兵值**：队列没有“关闭”概念，结束信号靠约定一个特殊值（如 `None`）；几个消费者就要放几个哨兵。
- **`maxsize` 是背压**：队列满了生产者阻塞，天然限流；无界队列在生产快、消费慢时会把内存吃爆。

## 4. 线程池：concurrent.futures

日常多线程优先用 `ThreadPoolExecutor`，别裸 `Thread` 撒出去不管：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

urls = [...]

with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(fetch, u) for u in urls]
    for f in as_completed(futures):
        result = f.result()  # 任务里抛的异常在这里才抛出来
```

- **异常去哪了**：任务里抛的异常不会在 `submit` 时出现，而是存在 `Future` 里，调 `result()` 时才重新抛出。不调 `result()`，异常就静默丢了——批量任务收尾必须逐个 `result()` 或 `as_completed` 过一遍。
- **map vs submit**：`pool.map` 按提交顺序出结果，适合“结果顺序要对上输入”；`submit` + `as_completed` 按完成顺序出结果，适合“谁先完先处理谁”。
- **`max_workers` 默认值**：`min(32, os.cpu_count() + 4)`。IO 密集可适当调大，但上限受对端服务承受力制约，不是越大越好；`with` 块退出时自动 `shutdown(wait=True)`，等全部任务跑完才往下走。

## 5. 常见坑

- **竞态**：`counter += 1` 是读、改、写三步，两个线程交错就会互相覆盖更新。共享可变状态要么加锁，要么只通过 `Queue` 传递、不要共享。
- **守护线程写一半被杀**：主线程一退出，守护线程当场掐断，落盘、写库、发请求都可能只做一半。不能丢的收尾放主线程或非守护线程，并 `join()` 确认。
- **忘记 join / 不 shutdown**：主线程跑完、非守护线程还在跑，进程迟迟不退出；线程池不用 `with` 包起来，`submit` 完主线程直接退出也一样拖住。生命周期永远显式收口。
- **线程里混 asyncio**：事件循环不是线程安全的，别在线程里直接调另一个线程的 loop 方法；跨线程递任务用 `asyncio.run_coroutine_threadsafe`，或干脆让 asyncio 侧用 `run_in_executor` 把阻塞活扔给线程池。
- **锁里调外部回调**：持锁期间调不知道会干什么的回调（钩子、用户函数），回调里再拿同一把锁就是死锁，回调耗时还拖住所有等锁的线程。锁内只做最小必要的状态修改。

## 6. GIL 边界与选型

GIL 的完整机制见 [Python 多进程与 GIL](开发语言/Python/Python-多进程与GIL.md)，线程篇只记边界结论：

| 场景 | 线程表现 | 该用什么 |
|---|---|---|
| 调接口、等数据库、读写文件、sleep | 有效，阻塞时释放 GIL | 线程池 / asyncio |
| 纯 Python 循环、字符串处理、自研算法 | 占着 GIL，多线程跑不满多核，甚至比单线程慢 | 进程池 |
| NumPy 大数组这类会释 GIL 的 C 扩展 | 可能吃到多核 | 逐库实测确认，别默认 |

粗口诀沿用 GIL 篇的：**等外部的用线程/协程，算自己的用进程**。线程、asyncio、进程三者的选型对照见 GIL 篇第 2 节的表，本篇不再重复。

## 7. 面试与自测

1. 线程间共享一个计数器做 `+=`，为什么结果会少？怎么修？
2. Lock 和 RLock 的区别？什么场景必须用 RLock？
3. 生产者消费者为什么要用 Queue 而不是 list + Lock 自己拼？
4. ThreadPoolExecutor 里任务抛了异常，去哪了？怎么拿到？
5. CPU 密集任务用线程池为什么没用？线程池在什么情况下反而比单线程慢？

### 参考答案

1. `+=` 是读、改、写三步，两个线程交错时一个线程的写会被另一个覆盖，结果比预期少。修法：用 `Lock` 包住读改写（推荐 `with lock`，异常也不漏放），或不共享计数器、改走 `Queue` 传递结果由一方汇总。
2. Lock 不可重入，同一线程第二次 `acquire` 会把自己锁死；RLock 允许同一线程多次获取，释放次数要配对。必须用 RLock 的场景是递归函数、或持锁方法调用另一个也要持锁的方法；普通场景用 Lock 即可，RLock 会掩盖锁粒度过大的设计问题。
3. `queue.Queue` 本身线程安全，`put`/`get` 自带锁；`maxsize` 提供背压，满了生产者阻塞，不丢数据；配 `task_done()`/`join()` 能等全部任务完成。自己用 list + Lock 拼要重做这三件事，还容易漏调 `task_done()`、忘处理满队列，等于重造一个更差的轮子。
4. 异常存在 `Future` 里，`submit` 时不抛，调 `result()` 时才重新抛出；不调 `result()` 异常就静默丢失。收尾必须逐个 `result()` 或用 `as_completed` 过一遍；`pool.map` 会在迭代到对应结果时抛出。
5. 纯 Python 计算全程占着 GIL，多线程同一时刻只有一个在跑字节码，跑不满多核，还多了切换开销。线程池反而更慢的情况：任务本身是 CPU 密集且粒度小，切换 + 锁开销盖过并行收益；这时该用进程池，或先确认是不是能释 GIL 的 C 扩展计算。

## 8. 关联知识

- [Python 多进程与 GIL](开发语言/Python/Python-多进程与GIL.md)：GIL 机制全解、三种并发模型选型表、进程池用法。
- [asyncio 并发模型](开发语言/Python/asyncio.md)：高并发 IO 的单线程方案，线程数上千量级时的替代选择。
- [Python](开发语言/Python/Python.md)：语言基础与常见坑总览。
