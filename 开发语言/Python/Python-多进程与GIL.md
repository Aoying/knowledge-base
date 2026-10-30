---
知识点: Python 多进程与 GIL
层级: 进阶
适用版本: Python 3.11+
更新日期: 2026-10-31
关联:
  - asyncio
  - Python
标签:
  - Python
  - 并发
  - 原理
---

# Python 多进程与 GIL

## 1. 核心概念：GIL 是什么

CPython 的对象引用计数靠一把全局解释器锁（Global Interpreter Lock，GIL）保护：**同一进程内，任一时刻只有一个线程在执行 Python 字节码**。线程切换由解释器周期性触发（默认每 5 毫秒检查一次，`sys.setswitchinterval` 可调）。

为什么不去掉：引用计数本身不是线程安全的，去掉 GIL 要给每个对象加锁或换 GC 机制，单线程性能会明显变差，C 扩展生态也要大改。这是二十多年的历史包袱，理解它的边界比抱怨它更重要：

- GIL 只锁**字节码执行**。线程在做阻塞 IO（网络、文件、sleep）时会主动释放 GIL，所以 IO 密集场景多线程完全有效。
- 纯 Python 的 CPU 计算（循环、字符串处理、自研算法）全程占着 GIL，多线程跑不满多核，甚至因切换开销比单线程还慢。
- 部分 C 扩展在计算时会释放 GIL（如 NumPy 的大数组运算），这类库多线程能吃到多核——但要逐库确认，不能默认。

## 2. 三种并发模型选型

| 模型 | 适合 | 不适合 | 说明 |
|---|---|---|---|
| threading | IO 密集：调接口、等数据库、爬取 | 纯 Python CPU 计算 | 最轻量，共享内存改数据方便，但要自己管锁 |
| asyncio | 高并发 IO：成百上千长连接、流式调用 | 混入同步阻塞调用 | 单线程协程，见 [asyncio 并发模型](开发语言/Python/asyncio.md) |
| multiprocessing | CPU 密集：解析、计算、编码、批量文本处理 | 任务粒度太细、数据量巨大 | 真并行吃多核，代价是进程间要序列化传数据 |

粗口诀：**等外部的用线程/协程，算自己的用进程**。混合场景常见 asyncio 主框架 + 把 CPU 段扔进进程池（`loop.run_in_executor` 接 `ProcessPoolExecutor`）。

## 3. multiprocessing 要点

### 3.1 起进程的两种方式

- `fork`（Linux 默认）：子进程直接复制父进程内存，开销小；但父进程里有线程、锁、连接时复制过去就是雷区（锁可能正被别的线程持有，复制后永远解不开）。
- `spawn`（Windows 默认、macOS 默认）：子进程是全新解释器，重新 import 主模块再执行入口。干净但启动慢，且**主模块顶层代码会被重新执行**。
- 规范动作跨平台只有一条：入口必须写在 `if __name__ == "__main__":` 守卫里，否则 spawn 下会递归起进程直接报错。这也是 Windows 上写多进程脚本最常见的翻车点。

### 3.2 现成的池子

日常优先用 `concurrent.futures.ProcessPoolExecutor`，接口和线程池一致：

```python
from concurrent.futures import ProcessPoolExecutor

def analyze_chapter(path: str) -> dict:
    # CPU 密集的纯计算：解析文本、统计、切分（函数必须定义在模块顶层，可 pickle）
    ...

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as pool:
        for result in pool.map(analyze_chapter, chapter_paths):
            save(result)
```

`multiprocessing.Pool` 功能更全（apply_async、回调），但要自己管 close/join 生命周期，新代码没特殊需求用 concurrent.futures 版。

### 3.3 数据怎么过去

进程之间不共享内存（默认），任务参数和返回值都要 **pickle 序列化**过一遍进程边界：

- 能 pickle 的：基础类型、模块顶层定义的类与函数、dataclass 等常规对象。
- 不能 pickle 的：lambda、闭包、局部定义的类/函数、打开的文件句柄、数据库连接、生成器。这类对象塞进任务参数会在提交时直接抛 `Can't pickle`——全库疑难索引里 Python 的那条经典报错就是它。
- 大对象别当参数反复传：每传一次序列化一次。数据大的做法是子进程自己按路径/ID 去读，或用共享内存（`multiprocessing.shared_memory`，3.8+）传字节块，管理成本明显更高，非必要不上。

## 4. 项目落点：小说引擎的批量任务怎么分

ao 的小说创作引擎里有两类批量活，正好落在选型表的两边：

- **调 Ollama 抽取章节信息**：瓶颈在等本地模型返回，是 IO 密集。用 asyncio 或线程池并发发请求即可，开进程纯属浪费——进程池帮不了等网络的忙。并发度受限于 Ollama 本身的并行处理能力，先小并发试再往上加。
- **本地文本处理**：分章切分、清洗、统计、必要时本地 embedding 前的预处理，是 CPU 密集。这类才上进程池，`max_workers` 从物理核数起步、结合内存余量（每进程各加载一份数据/模型时内存是先爆的那个）实测调整。
- 两段式流水线常见组合：进程池做完 CPU 预处理，结果回主进程后走 asyncio 批量调模型，不要在子进程里再套一套并发模型，排查问题时分不清是谁的锅。

## 5. 疑难杂症

- **子进程起了但永远不结束**：父进程异常退出没 join、或子进程在等一把 fork 时复制过来的死锁。用 concurrent.futures 上下文管理器托管生命周期，关键路径加超时，不要裸 `Process.start()` 撒出去不管。
- **Windows 上一跑就递归爆栈/重复执行**：入口没放进 `__main__` 守卫，spawn 重新 import 主模块时又触发了起进程代码。
- **Pool 里再开 Pool**：守护进程（daemon）不允许有子进程，池里的 worker 是守护进程，在任务函数里再建池直接抛错。需要嵌套并行时改架构：拍平任务粒度，或让内层降级为线程/顺序执行。
- **结果对得上但比单进程还慢**：任务粒度太细，序列化 + IPC 的开销盖过计算收益。用 `chunksize` 把小任务打包提交，或干脆合并任务。

## 6. 常见误区

- 误区：GIL 意味着 Python 多线程没用。IO 线程是真实有效的，Web 请求、数据库等待场景里多线程一直是主力方案之一。
- 误区：多进程一定快。进程启动、序列化、结果回传都是成本，任务小于几十毫秒量级时先算总账。
- 误区：进程之间能像线程一样共享变量。默认完全不共享，`Manager` 和 `Value/Array` 是通过代理/共享内存模拟出来的，每次访问都有 IPC 开销，别拿它当普通变量高频读写。
- 变化：CPython 3.13 起提供实验性的无 GIL 构建（free-threading），但非默认、需专门构建，第三方 C 扩展兼容性仍在补齐中。现阶段生产与学习口径都按「GIL 存在」理解，不要把选型押在它身上。

## 小技巧与冷知识

【小技巧】每个 worker 只加载一次大资源，别当任务参数反复传：用 `ProcessPoolExecutor(initializer=init, initargs=(...))` 或 `Pool(initializer=...)`，在 worker 启动时加载一次模型、词典、连接，存在 worker 的全局变量里复用。把大对象塞进每个任务的参数，等于每跑一个任务就序列化、反序列化一次，初始化函数的成本只付一次，参数传递的成本要付 N 次。

【小技巧】结果量大时别用 `pool.map`，用 `imap` / `imap_unordered` 流式取结果：`map` 会把全部结果攒成一个 list 再返回，十万条结果先在内存里堆满；`imap` 返回迭代器，边算边取，内存里只留当前这一批。任务耗时不均时用 `imap_unordered`，它按完成顺序回结果，不会被最慢的那个任务堵住整条输出。

【小技巧】想“谁先算完先处理谁”用 `as_completed`：`concurrent.futures.as_completed(futures)` 按完成顺序吐 future，快的先回、慢的后回，适合“先出结果先落库、先展示”的场景。`pool.map` 是按提交顺序回结果的，最慢的任务没回来之前，后面算好的结果也只能等着，批量任务里这是最常见的隐性等待。

【小技巧】worker 用久了会脏：内存慢慢涨、缓存越来越旧、全局状态被前面的任务改乱。`Pool(maxtasksperchild=N)` 和 `ProcessPoolExecutor(max_tasks_per_child=N)` 让每个 worker 干满 N 个任务就重启换新，代价是一次进程启动，换来的是状态清零。长跑的批量任务里，这比在任务函数里手动清理全局变量可靠得多。

【小技巧】父进程已有线程、锁、连接，又想用 fork 的便宜，选 `forkserver`：`forkserver` 启动方式先起一个干净的 server 进程，子进程都从它 fork，不复制父进程的线程和锁状态。它比纯 fork 安全（避开了复制死锁的雷区），比 spawn 快（不用重新 import 整个模块），是 Linux 上“父进程已经很脏了”时的折中方案，`multiprocessing.get_context("forkserver")` 就能拿到。

【小技巧】`set_start_method` 只能在任何进程创建前调一次，调晚了直接报 `RuntimeError`：启动方式是进程级的全局设置，一旦起过一个进程就锁死了。想在 Linux 上强制用 spawn 又不想改全局，用 `multiprocessing.get_context("spawn").ProcessPoolExecutor(...)` 只影响这一个池子，别的库照常用默认方式，互不干扰。

【小技巧】Jupyter 或交互式环境里起进程池，常在提交时报 `Can't pickle`：交互式里定义的函数没有可 import 的模块路径，pickle 按名字找不到它。解决办法是把任务函数写进一个 `.py` 文件再 `import` 进来，让子进程能按模块名重新找到它；同样的函数放进脚本文件跑就没事，问题不在代码逻辑，在“函数住在哪”。

【小技巧】Ctrl+C 时 `Pool` 的 worker 默认忽略 SIGINT，只有主进程能收到 `KeyboardInterrupt`：这是设计好的——子进程不抢中断信号，主进程统一处理。主进程里要写 `try/except KeyboardInterrupt` 再调 `pool.terminate()`，否则主进程退了、worker 可能还留在后台继续算；`ProcessPoolExecutor` 的上下文管理器在退出时也会等 worker 收尾，异常路径别绕过它。

【小技巧】fork 之后“共享内存”是假象：子进程靠写时复制（COW）和父进程共享页面，但 CPython 的引用计数本身会写内存——对象被引用一次、计数就加一次，页面一碰就复制。父进程预加载的大数据并不能免费共享给子进程，复制成本只是被摊薄了。真要零拷贝共享大数组，用 `multiprocessing.shared_memory` 配 `numpy.frombuffer`，别指望 fork 白送。

【小技巧】`max_workers` 别直接照搬 `os.cpu_count()`：容器里 `cpu_count()` 看到的是宿主机的核数，进程实际能用的可能少得多，超配只会让进程互相抢核。Linux 上用 `len(os.sched_getaffinity(0))` 看当前进程的 CPU 亲和性，Python 3.13 起还有 `os.process_cpu_count()`，直接返回“这个进程能用的核数”，按它定 worker 数才贴合容器限额。

【冷知识】macOS 上默认启动方式在 Python 3.8 就从 fork 改成 spawn 了：fork 一个多线程的父进程在 macOS 上频频出问题（锁、Objective-C 运行时状态复制过去都是雷），CPython 干脆在 3.8 把 macOS 的默认值改成了 spawn。同一份代码在 Linux 上默认 fork、在 macOS 上默认 spawn，跨平台跑多进程时启动方式不是“写死”的，是跟着平台走的。

【冷知识】无 GIL 的路线来自 Sam Gross 的 nogil 工作，2023 年被接受为 PEP 703：他先在 2021 年做出概念验证，证明去掉 GIL 后单线程性能损失可以控制在可接受范围，2023 年 PEP 703 正式被采纳，Python 3.13 才有了实验性的 free-threading 构建。GIL 不是“没人想动”，而是动了三十年才找到一条性能账算得过来的路。

【冷知识】`os.register_at_fork`（Python 3.7+）是官方给 fork 埋雷准备的补丁：它能在 fork 之后、子进程继续执行之前跑一段清理函数，比如重建一把被复制过来的锁、重新打开一个失效的连接。它的存在本身就说明了 fork 的风险是真实存在的——连标准库都要给它留一个“fork 后自救”的钩子。

## 7. 面试与自测

- GIL 锁的到底是什么？为什么 IO 密集多线程不受它拖累？
- fork 与 spawn 的差别？为什么 Windows 上必须用 `__main__` 守卫？
- 哪些对象不能被 pickle？报错出现在哪一步？
- CPU 密集任务为什么用线程池没用、进程池有用？
- 任务粒度太细时，进程池会慢在哪几个环节？

### 参考答案

1. GIL 锁的是同一进程内 Python 字节码的执行：任一时刻只有一个线程在执行 Python 字节码，线程切换由解释器周期性触发（默认每 5 毫秒检查一次）。IO 密集多线程不受拖累，是因为线程在做阻塞 IO（网络、文件、sleep）时会主动释放 GIL；纯 Python 的 CPU 计算全程占着 GIL，才跑不满多核。
2. fork 是 Linux 默认：子进程直接复制父进程内存，开销小，但父进程里的线程、锁、连接复制过去就是雷区（锁可能正被别的线程持有，复制后永远解不开）。spawn 是 Windows 与 macOS 默认：子进程是全新解释器，重新 import 主模块再执行入口，干净但启动慢。Windows 上必须用 __main__ 守卫，是因为 spawn 会重新执行主模块顶层代码，入口不包进守卫就会递归起进程直接报错。
3. 不能被 pickle 的有：lambda、闭包、局部定义的类与函数、打开的文件句柄、数据库连接、生成器。报错出现在提交时：任务参数和返回值都要 pickle 序列化过一遍进程边界，这类对象塞进任务参数会在提交时直接抛 Can't pickle，而不是在子进程执行时才暴露。
4. 因为纯 Python 的 CPU 计算全程占着 GIL，线程池里的线程再多也只能串行执行字节码，甚至因切换开销比单线程还慢。进程池有用是因为每个进程有自己独立的 GIL，能真并行吃多核，代价是进程间要序列化传数据。口诀是等外部的用线程或协程，算自己的用进程。
5. 慢在三个环节：进程启动开销、任务参数与返回值的 pickle 序列化、结果回传的 IPC 开销；任务小于几十毫秒量级时这些开销会盖过计算收益。缓解办法是用 chunksize 把小任务打包提交，或干脆合并任务粒度；另外大对象别当参数反复传，每传一次序列化一次，数据大时让子进程自己按路径或 ID 去读。
## 8. 关联知识

- [asyncio 并发模型](开发语言/Python/asyncio.md)：高并发 IO 的单线程方案，与本篇组成 Python 并发选型的两半。
- [Python](开发语言/Python/Python.md)：语言基础与常见坑总览。
