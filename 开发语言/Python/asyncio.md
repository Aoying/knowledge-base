---
知识点: asyncio 并发模型
层级: 进阶
适用版本: Python 3.11+
更新日期: 2026-10-31
关联:
  - Python
  - Python知识点地图
  - pydantic
  - dataclass与typing
标签:
  - Python
  - 并发
  - 实践
---

# asyncio 并发模型

## 1. 核心概念

asyncio 是单线程协作式并发：一个事件循环（event loop）在一个线程里轮流推进成百上千个协程，协程只在 `await` 处主动让出——让出时通常是在等 IO（网络、磁盘、定时器）。没有抢占、没有锁竞争，代价是**任何一段不让出的同步代码都会卡住所有协程**。

判断标准只有一条：**瓶颈在等 IO 就用 asyncio，在算 CPU 就用多进程/多线程**。

- 小说引擎批量抽取：每章要等 Ollama 返回（几秒到几十秒），是教科书级的 IO 密集，asyncio 一条线程就能挂几十个在途请求。
- 反例：本地算 embedding、批量分词、跑向量检索的计算段是 CPU 密集，塞进协程只会互相卡，要么放线程池/进程池，要么交给专门的服务进程。

## 2. 基础知识（小说引擎批量抽取章节时用）

```python
import asyncio, httpx

async def extract_one(client, chapter_id):
    r = await client.post("http://127.0.0.1:11434/v1/chat/completions",
                          json={"model": "qwen3:14b", "messages": [...]})
    return chapter_id, r.json()

async def main(ids):
    async with httpx.AsyncClient(timeout=120) as client:
        sem = asyncio.Semaphore(4)  # Ollama 并发别打太高，4 左右
        async def bounded(i):
            async with sem:
                return await extract_one(client, i)
        return await asyncio.gather(*(bounded(i) for i in ids))

asyncio.run(main(range(1, 51)))
```

- IO 密集型（调 Ollama API）用 asyncio，一个线程顶几十个并发；CPU 密集型才上多进程。
- `Semaphore` 限流是保命项：Ollama 默认并发一高就排队甚至超时。
- 结构化输出别忘了 `response_format={"type":"json_object"}`，省得正则去抠。

几个基本件：

- `async def` 定义协程函数，调用它只得到协程对象、并不执行，必须 `await` 或包装成 Task 才跑起来——忘了 await 是新手第一坑，运行时会警告 `coroutine was never awaited`。
- `asyncio.create_task(coro)` 把协程注册成 Task 立即排期，适合"先发出去、稍后再收结果"的场景；记得保留 Task 引用（被 GC 回收会静默中断）。
- `asyncio.run(main())` 是程序入口：建事件循环、跑完、收尾，一个进程只调一次。

## 3. 并发控制：gather、超时与取消

- `asyncio.gather(*aws)`：并发跑、按传入顺序返回结果列表；任一抛异常默认整体抛出，其余任务继续跑但结果丢弃。批量抽取更想要"谁成功算谁"，用 `return_exceptions=True` 把异常当结果收回来，逐个归类处理（成功落库/失败重试/校验不过回喂重抽）。
- `asyncio.TaskGroup`（3.11+）：结构化并发，组内任一失败会取消其余并抛 `ExceptionGroup`。适合"同生共死"的任务组（如一章的抽取+校验+落库），新代码优先它，少写手工收尾。
- **超时**：`async with asyncio.timeout(120):`（3.11+）或旧的 `asyncio.wait_for`。超时本质是取消任务，被取消方会收到 `CancelledError`——所以 `except Exception` 兜底时**不要吞掉 CancelledError**（3.8+ 它已不继承 Exception，但自己写 `except BaseException` 或裸 `except:` 仍会误吞）。重试逻辑要把"超时取消"和"业务异常"分开计数。
- **取消传播**：`task.cancel()` 只是请求取消，对方在下一个 await 点才真正停下；正在跑的同步代码段拦不住。批量任务关停时给每个任务发 cancel 后再统一 await 收尾，别直接杀进程丢在途写入。

## 4. 异步管道里的限流与背压

Semaphore 只是第一层。真实管道（批量抽取 600 章）还要考虑：

- **生产者-消费者用 `asyncio.Queue`**：生产者按章节入队，N 个 worker 协程消费，队列 `maxsize` 形成天然背压——不限制队列长度时，600 章的提示词会先把内存吃满。
- **重试退避**：Ollama 超时/返回坏 JSON 时别立即重试打爆服务，指数退避 + 抖动（如 1s、2s、4s 上限封顶）；重试次数单独计数，超过阈值的章节进失败清单，跑完统一汇报，不要让一章坏数据拖死整批。
- **连接复用**：`httpx.AsyncClient` 全程复用一个实例（连接池），不要每章 new 一个；同步版 `requests` 在协程里是阻塞调用，绝对不能混用——同步 SDK 要么换异步版，要么 `asyncio.to_thread` 丢线程池。

```python
async def worker(queue, client, results):
    while True:
        cid = await queue.get()
        try:
            results[cid] = await extract_one(client, cid)
        except Exception as e:
            results[cid] = e          # 失败也记录，跑完统一统计
        finally:
            queue.task_done()
```

## 5. 与同步世界打交道

- **SQLite 写入**：标准库 sqlite3 是同步阻塞的。单章写入是毫秒级、影响有限；但批量落库时几千次写挤在事件循环里会明显卡顿。可选：写入集中到一个消费者协程串行执行（顺带解决 SQLite 并发写锁问题）、或用 `aiosqlite`、或老实把整批落库 `to_thread`。
- `asyncio.to_thread(func, *args)`（3.9+）：把阻塞函数丢默认线程池并 await 结果，是包住老式同步代码（文件 IO、同步 HTTP、CPU 不重的计算）的标准办法。
- CPU 密集段（本地 embedding 计算、大 JSON 解析量极大时）用 `ProcessPoolExecutor` + `loop.run_in_executor`，别指望线程池绕开 GIL。

## 6. 常见误区

- **在协程里写同步阻塞调用**：`time.sleep`、`requests.post`、大文件读写——一个卡住，全线卡住。症状是"开了 50 并发，吞吐跟单线程一样"，先查这个；换异步库或 `to_thread`。
- **gather 不加限流一口气放一万个任务**：连接数、内存、对方服务一起爆。并发上限永远要显式设（Semaphore / 固定 worker 数）。
- **异常被 gather 吞没或一个失败全批作废**：见第 3 节，批量任务用 `return_exceptions=True` 或逐任务 try 收集。
- **以为 asyncio 能加速 CPU 任务**：协程再多也是一条线程轮流算，总时长不变还多调度开销；GIL 之下多线程同理，CPU 密集只有多进程能真正并行。

## 小技巧与冷知识

【小技巧】同一个协程对象只能 await 一次：`c = extract_one(...)` 之后 `await c` 两次，第二次直接报 `RuntimeError: cannot reuse already awaited coroutine`。想跑两次就调两次协程函数、每次拿到新对象；把协程对象存进列表、循环里反复 await，是“协程复用”这个坑最常见的写法。

【小技巧】任务里抛了异常又没人收，错误只会变成一条警告：Task 抛异常后没人 `await` 它、也没人调 `result()`，解释器只在 GC 时打一条 `Task exception was never retrieved`，程序照常往下跑。批量任务里每个 Task 都要有人收尾——`gather`、`TaskGroup`、或者 `add_done_callback` 里调 `result()` 触发异常；没人收的 Task 等于把错误静默丢掉。

【小技巧】给 Task 起个业务名，排障时堆栈里直接能读出来：`asyncio.create_task(extract_one(...), name="extract-12")`，日志和 `asyncio.all_tasks()` 里看到的就不是一串匿名的 `Task-37`，而是“哪一章的抽取卡住了”。调试模式下（`PYTHONASYNCIODEBUG=1` 或 `loop.set_debug(True)`）还会把执行超过 100 毫秒的回调打出来，慢回调的定位先开 debug 再猜。

【小技巧】`asyncio.run()` 一个进程只调一次，在已有事件循环里调它直接报错：在 Jupyter、FastAPI 端点、别的异步框架里再调 `asyncio.run(main())`，会报 `RuntimeError: asyncio.run() cannot be called from a running event loop`。已经在协程里时直接 `await` 就行；实在要在同步代码里从另一个线程提交协程，用 `asyncio.run_coroutine_threadsafe(coro, loop)`，它返回一个 `concurrent.futures.Future`，在别的线程里调 `.result()` 等结果。

【小技巧】只想“等其中一个先完成”别用 `gather`，用 `asyncio.wait`：`wait(tasks, return_when=asyncio.FIRST_COMPLETED)` 返回已完成和未完成两组任务，适合“多个源抢首个响应、剩下的取消”这类竞速场景。`gather` 是“全跑完再一起收”，`wait` 是“按完成情况分批收”；竞速时用 `gather` 会被最慢的那个拖住，`wait` 拿到第一个就能先处理、再把剩下的 `cancel()` 掉。

【小技巧】取消时 `finally` 里的 `await` 会再次触发取消：任务被取消后进入 `finally` 做收尾，里面再 `await` 写库、关连接，`CancelledError` 会在第一个 `await` 点重新抛出来，收尾做一半就断了。收尾里必须 `await` 的那几步用 `asyncio.shield(...)` 包起来，让它不受外层取消影响跑完；或者把收尾写成同步代码，别在取消路径上再 `await`。

【小技巧】“超时取消但落库必须完成”也用 `shield`：`await asyncio.wait_for(asyncio.shield(save_result(...)), timeout=120)` 里，超时只取消外层的等待，内层的保存任务照常跑完。`shield` 的语义是“保护内层任务不被外层取消波及”，不是“不超时”——超时照样报 `TimeoutError`，只是被保护的操作不会半路夭折；落库、发确认、记账这类“做了就必须做完”的操作才配用它，普通查询别滥用。

【小技巧】Semaphore 拿了就要放，`BoundedSemaphore` 会帮你查账：普通 `Semaphore` 多 `release()` 一次不会报错，计数悄悄变大，限流就漏了；`BoundedSemaphore` 超额释放直接抛 `ValueError: BoundedSemaphore released too many times`。限流信号量优先用有界版，超额释放是 bug 不是“多放了一次”，让它在开发期就炸出来比线上限流失效强得多。

【小技巧】`asyncio.sleep(0)` 是协程里最轻的让步点：它不真睡，只是把控制权交回事件循环、让别的协程先跑一轮。长循环里全是纯 Python 计算、中间没有 `await` 时，别的协程会被饿很久，插一句 `await asyncio.sleep(0)` 就能让出；它和 `time.sleep(0)` 不是一回事，后者会阻塞整个事件循环。

【小技巧】全局未处理异常用事件循环的异常处理器集中收：`loop.set_exception_handler(handler)` 能收到所有没人接的 Task 异常、回调里的异常，适合在批量管道入口统一记日志、发告警。别只靠 `try/except` 包每个任务——漏包一个，异常就只剩 `Task exception was never retrieved` 一条警告；集中处理器是最后一道网，单任务的 `try` 是第一道。

【小技巧】`asyncio.Queue` 的 `get_nowait` / `put_nowait` 抛的是 `QueueEmpty` / `QueueFull`，不是返回 `None`：非阻塞取数要包 `try/except asyncio.QueueEmpty`，直接判 `if not queue.empty()` 再取是竞态——判断和取数之间别的协程可能先取走。`empty()` / `full()` 只适合做监控指标，不适合做分支条件。

【小技巧】`asyncio.Event` 是“一对多通知”，`asyncio.Condition` 是“带条件的等待”：一个事件发生要叫醒所有等待者时用 `Event.set()`；生产者-消费者要“等到队列非空再取”时用 `Condition` 配 `while` 循环判条件。`Event` 没有计数语义，`set()` 之前就 `wait()` 的协程和之后才 `wait()` 的协程行为不一样——先 `set()` 再 `wait()` 的直接通过，别把它当信号量用。

【冷知识】asyncio 是 Python 3.4 随 PEP 3156 引入的，由 Guido van Rossum 主导设计：`async`/`await` 语法是 3.5（PEP 492）才来的，`asyncio.run` 是 3.7、`TaskGroup` 和 `asyncio.timeout` 是 3.11。也就是说今天这套写法是三个大版本迭代出来的，3.4 时代的 `@asyncio.coroutine` + `yield from` 写法早已废弃，看到老代码里 `yield from` 就知道那是 3.5 之前的风格。

【冷知识】Windows 上默认事件循环在 3.8 换成了 ProactorEventLoop：3.8 之前 Windows 默认是 SelectorEventLoop，子进程、信号这些能力受限；3.8 起默认用基于 I/O 完成端口（IOCP）的 Proactor 实现。同一份 asyncio 代码在 Linux 和 Windows 上跑的不是同一个循环实现，跨平台出问题时先问“这是在哪个循环上”，而不是先怀疑业务逻辑。

【冷知识】uvloop 是 asyncio 事件循环的一个可替换实现：它是基于 libuv 的 C 扩展，uvicorn 默认就用它。用法是 `asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())`，业务代码不用改一行——事件循环是可插拔的，`asyncio` 只定义了协程、Task、Future 这套接口，底下的循环实现可以换。

## 面试与自测

1. 本篇判断“用不用 asyncio”的标准只有一条，是什么？结合小说引擎批量抽取说明正反两个例子。
2. 批量抽取时，`asyncio.gather` 默认的异常行为有什么问题？应该换成什么做法？
3. 超时取消时，为什么重试逻辑要把“超时取消”和“业务异常”分开计数？兜底 except 时要注意什么？

### 参考答案

1. 标准是：瓶颈在等 IO 就用 asyncio，在算 CPU 就用多进程/多线程。正例是小说引擎批量抽取，每章要等 Ollama 返回几秒到几十秒，是教科书级的 IO 密集，一条线程就能挂几十个在途请求；反例是本地算 embedding、批量分词、跑向量检索的计算段，塞进协程只会互相卡。
2. 默认行为是任一任务抛异常就整体抛出，其余任务继续跑但结果丢弃，等于一章失败整批作废。批量抽取更想要“谁成功算谁”，用 `return_exceptions=True` 把异常当结果收回来，逐个归类处理：成功落库、失败重试、校验不过回喂重抽。
3. 超时本质是取消任务，被取消方会收到 `CancelledError`；把它和业务异常混在一起计数，重试策略就失真了。兜底时不要吞掉 `CancelledError`：3.8+ 它已不继承 Exception，但自己写 `except BaseException` 或裸 `except:` 仍会误吞。

## 7. 关联知识

- [pydantic](开发语言/Python/pydantic.md)：每章抽取结果返回后先过校验，校验失败回喂模型重抽，是管道的标准收尾
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：管道内传递的章节/结果结构用 dataclass 建模
- [装饰器](开发语言/Python/装饰器.md)：重试退避常用装饰器封装

## 8. 一句话总结

asyncio 是"等 IO 的时候别闲着"，不是"算得更快"：限流、超时、失败收集三件事做齐，批量管道才算生产可用。
