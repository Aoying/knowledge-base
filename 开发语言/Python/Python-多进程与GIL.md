---
知识点: Python 多进程与 GIL
层级: 进阶
适用版本: Python 3.11+
更新日期: 2026-10-23
关联:
  - asyncio 并发模型
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

## 7. 面试与自测

- GIL 锁的到底是什么？为什么 IO 密集多线程不受它拖累？
- fork 与 spawn 的差别？为什么 Windows 上必须用 `__main__` 守卫？
- 哪些对象不能被 pickle？报错出现在哪一步？
- CPU 密集任务为什么用线程池没用、进程池有用？
- 任务粒度太细时，进程池会慢在哪几个环节？

## 8. 关联知识

- [asyncio 并发模型](开发语言/Python/asyncio.md)：高并发 IO 的单线程方案，与本篇组成 Python 并发选型的两半。
- [Python](开发语言/Python/Python.md)：语言基础与常见坑总览。
