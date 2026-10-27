---
知识点: Python-日志与HTTP客户端
层级: 进阶
适用版本: Python 3.10+（requests 2.x / httpx 0.x）
更新日期: 2026-10-24
关联:
  - Python
  - Logback与日志规范
  - asyncio
  - SQLAlchemy
标签:
  - Python
  - 实践
  - 排错
---

# Python 日志与 HTTP 客户端

## 1. 核心概念

上半讲 `logging`，下半讲 HTTP 客户端——小说引擎最常打交道的两个基础设施：前者决定出问题时有没有线索，后者是引擎与本地 Ollama 之间的唯一通道。两者共同的毛病都是「默认能跑但不设防」：日志默认无格式无落盘，requests 默认无超时，一出问题就现形。

与 Java 侧对位一句：logging 的 logger 层级、handler、formatter 对应 [Logback 与日志规范](工程实践/Logback与日志规范.md) 的 logger/appender/layout，概念一一对应，只是 Python 没有强制的门面层。

## 2. logging 五件套

- **Logger**：按点分层级命名（`novel.chapter` 是 `novel` 的子 logger），业务代码里统一 `logger = logging.getLogger(__name__)`，级别沿层级继承。
- **Level**：DEBUG < INFO < WARNING < ERROR < CRITICAL。库代码只打日志、**不设级别不加 handler**（顶多挂 NullHandler），级别与输出去向由应用入口统一决定。
- **Handler**：决定日志去哪（控制台、文件、滚动文件、队列）。一个 logger 可挂多个。
- **Formatter**：决定长什么样。生产格式至少带时间、级别、logger 名，结构化场景可换 JSON formatter。
- **Filter**：做级别之外的细粒度过滤（如按请求 ID），用得少，知道有这层即可。

日志沿层级向父级**传播**（propagate 默认开）：子 logger 的日志会冒泡到父 handler。重复输出的经典根因就是自己挂了 handler 又让它冒泡到 root，打两遍。

## 3. 为什么不用 print，配置怎么写

print 没有级别（调不开、关不掉）、没有时间与上下文、无法按模块分流、轮转全靠手写。脚本跑通阶段无所谓，进了长流程批处理（逐章抽取跑几百章）就是灾难：出错时只有一句孤零零的输出，不知道哪章、哪一步。

配置用 `dictConfig` 一次到位，代码里只 `getLogger`：

```python
import logging
from logging.config import dictConfig

dictConfig({
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "std": {"format": "%(asctime)s %(levelname)s %(name)s: %(message)s"},
    },
    "handlers": {
        "console": {"class": "StreamHandler", "formatter": "std"},
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "engine.log", "maxBytes": 10 * 1024 * 1024,
            "backupCount": 5, "formatter": "std", "encoding": "utf-8",
        },
    },
    "loggers": {
        "novel": {"level": "INFO", "handlers": ["console", "file"], "propagate": False},
        "httpx": {"level": "WARNING"},          # 三方库日志单独压级
        "sqlalchemy.engine": {"level": "WARNING"},  # 打开就是 SQL 刷屏
    },
})
```

要点：消息用惰性参数 `logger.info("chapter %s done", ch_id)` 而不是 f-string——被级别过滤时 f-string 已先拼好，惰性参数则跳过拼接；记异常用 `logger.exception(...)` 或 `exc_info=True`，别把 traceback 拼进字符串。

## 4. HTTP 客户端：requests 与 httpx

| 维度 | requests | httpx |
|---|---|---|
| 同步 API | 主场 | 有，与 requests 风格高度相似 |
| 异步 API | 无（要另找 aiohttp） | 原生 `AsyncClient`，与 asyncio 协程同一套心智 |
| 底层 | urllib3 | h11/h2，自带连接池 |
| 迁移成本 | — | 同步代码改动很小 |

选型：纯脚本、同步流程用 requests 足够；抽取要并发调 Ollama 时走 httpx 的 `AsyncClient`（与 [asyncio](开发语言/Python/asyncio.md) 篇的并发模型接上），别在协程里塞同步 requests 堵死事件循环。

## 5. 超时三分法与连接池

**永远显式设超时**：requests 默认超时是无穷大，服务端 hang 住，客户端就永久挂起，批处理整夜卡死在一章上。

- requests 用元组拆两段：`timeout=(连接超时, 读取超时)`，如 `timeout=(3, 60)`。
- httpx 拆得更细：`Timeout(connect=3, read=60, write=10, pool=5)`——连接、读、写、等连接池空位四段。概念上就是连接/读/池三分（外加写）。
- 连接池：requests 用 `Session` + `HTTPAdapter(pool_connections, pool_maxsize)` 复用连接，别每次请求新建 Session；httpx 的 Client 本身即连接池，长期复用同一实例。

## 6. 重试策略

两条路线：

- **连接层重试**（urllib3 `Retry`，经 `HTTPAdapter(max_retries=...)` 注入 requests）：管连接错误与指定状态码，配 `backoff_factor` 做指数退避，尊重 `Retry-After`。

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=3, backoff_factor=0.5,
              status_forcelist=[429, 500, 502, 503, 504])
session.mount("http://", HTTPAdapter(max_retries=retry, pool_maxsize=10))
```

- **业务层重试**（tenacity）：`@retry(stop=stop_after_attempt(3), wait=wait_exponential(...), retry=retry_if_exception_type(...))`，按异常类型与返回结果决定重试，适合包在「调一次 LLM 抽取」这类业务函数上。

**重试边界**（比工具更重要）：只重试连接失败、超时与 5xx/429；4xx 是请求本身错了，重试纯属浪费。POST 等非幂等请求慎重试——服务端可能已处理成功只是响应丢了；要重试先确认有幂等键或操作可重入。

## 7. 落点：调本地 Ollama 的实战姿势

小说引擎经 `http://127.0.0.1:11434` 调本地模型，这类调用和调公网 API 的参数直觉完全不同：

- **read 超时按分钟给**：生成是慢工，首 token 延迟 + 长输出，read 超时 60 秒会把正常的大章生成误杀。按「连接 3 秒、读取数分钟」设，或直接走流式（下条），让超时只防真卡死。
- **流式读取**：`stream=True`（httpx 用 `client.stream(...)`）逐行读 NDJSON 增量，边收边落进度；卡住时能判断是模型在想还是连接已死，而不是等大响应干瞪眼。
- **失败重试边界**：连接被拒（Ollama 没起）可退避重试几次；**生成中途断流不要盲目整单重试**——会重复生成、还可能得到另一份结果。引擎的口径是整章抽取一个事务（见 [SQLAlchemy](开发语言/Python/SQLAlchemy.md)），失败就整章回滚重抽，重试发生在章节级而不是请求级。
- **代理坑**：机器上若设了 `http_proxy` 环境变量，requests/httpx 会把 `127.0.0.1` 也送去代理，本地服务莫名连不上。给本地地址设 `NO_PROXY=127.0.0.1,localhost`，或在 Session 上关代理（`trust_env=False`）。
- **日志联动**：调前打 INFO（章节号、模型名），失败打 ERROR 带状态码与响应前 200 字；httpx 自己的 logger 压到 WARNING，否则每次请求两行流水账刷屏。

## 8. 常见误区

- `basicConfig` 在 import 靠后时「不生效」：它只在 root 未配置时起作用，各处乱调必然互相覆盖，统一入口一处 dictConfig。
- 异常被 `except: pass` 吞掉又不打日志：批处理跑一夜，失败章全静默，比崩掉更难查。
- 重试不设上限与退避：对端故障时以最高频率捶打，本地服务也会被自己压垮。
- 把 API 响应原文整个打进日志：LLM 输出动辄数 KB，日志爆量还混敏感内容，截断再打。

## 9. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [Logback 与日志规范](工程实践/Logback与日志规范.md)：Java 侧日志体系，概念对位
- [asyncio](开发语言/Python/asyncio.md)：httpx 异步客户端的并发底座
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)：整章一个事务，失败回滚重抽
- [本地部署与 Ollama](AI与大模型/本地部署与Ollama.md)：被调用方的部署与参数
