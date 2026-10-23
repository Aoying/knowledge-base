---
知识点: asyncio 并发模型
层级: 进阶
适用版本: Python 3.11+
更新日期: 2026-10-23
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

## 7. 关联知识

- [pydantic](开发语言/Python/pydantic.md)：每章抽取结果返回后先过校验，校验失败回喂模型重抽，是管道的标准收尾
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：管道内传递的章节/结果结构用 dataclass 建模
- [装饰器](开发语言/Python/装饰器.md)：重试退避常用装饰器封装

## 8. 一句话总结

asyncio 是"等 IO 的时候别闲着"，不是"算得更快"：限流、超时、失败收集三件事做齐，批量管道才算生产可用。
