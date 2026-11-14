---
知识点: Python-日志与HTTP客户端
层级: 进阶
适用版本: Python 3.10+（requests 2.x / httpx 0.x）
更新日期: 2026-11-14
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

## 原理剖析

本节只把原文已有章节给出的结论拆开讲清机制：不引入新命令/参数，不补新数字，数字一律复用 §X 已核验的定量算例；凡原文没展开的点，文中明确标注，不作定论。

### 日志为什么会沿层级冒泡、又为什么打两遍

原文 §2 给出的机制是：日志沿层级向父级传播（propagate 默认开），子 logger 的日志会冒泡到父 handler。`novel.chapter` 是 `novel` 的子 logger，级别沿层级继承；一个 logger 可挂多个 handler。重复输出的经典根因就是自己挂了 handler 又让它冒泡到 root，打两遍——同一条记录先走自己挂的 handler 输出一次，再冒泡到父级 handler 又输出一次。本篇 dictConfig 里 `novel` logger 显式设了 `propagate: False`，就是把冒泡关掉，让输出只走自己挂的 console 与 file 两个 handler。库代码只打日志、不设级别不加 handler（顶多挂 NullHandler），级别与输出去向由应用入口统一决定——否则库与应用各配一套，冒泡路径上每级都可能再输出一次。

### 超时为什么要拆成连接、读取、写、等池位四段

原文 §5 给出的结论是：requests 用元组拆两段 `timeout=(连接超时, 读取超时)`，如 `timeout=(3, 60)`；httpx 拆得更细：`Timeout(connect=3, read=60, write=10, pool=5)`——连接、读、写、等连接池空位四段。拆段的原因是四段卡住的含义完全不同：连接超时 3 秒防的是“对端根本不通”，读超时防的是“连上了但响应迟迟不来”，等池位防的是“连接池满了、新请求排不上队”。不拆段、只给一个总超时，排障时就分不清卡在哪一段；拆开后，连接 3 秒就断说明网络或服务没起，读超时才触发说明服务在处理、只是慢。原文 §7 进一步收紧了读超时的量级：调本地 Ollama 时生成是慢工，首 token 延迟加长输出，read 超时 60 秒会把正常的大章生成误杀，应按「连接 3 秒、读取数分钟」设，或直接走流式，让超时只防真卡死。httpx 的默认超时只有 5 秒（原文小技巧节），直接拿默认 httpx 调 Ollama，5 秒就断——用 httpx 一定要显式传超时，别指望默认配置对本地大模型友好。

### 重试为什么只重试 5xx/429、不重试 4xx，POST 为什么要慎重

原文 §6 给出的重试边界是：只重试连接失败、超时与 5xx/429；4xx 是请求本身错了，重试纯属浪费。机制在于“失败能不能靠再试一次变好”：连接失败与超时是瞬时故障，重试可能成功；5xx 是服务端的问题，服务端恢复后重试可能成功；429 是被限流，退避后重试可能成功；4xx 是请求本身写错了，再试一百次还是同一个错。POST 等非幂等请求慎重试，是因为服务端可能已处理成功、只是响应丢了，重试等于把同一笔操作再执行一次。原文给出的两条路线分工不同：连接层重试（urllib3 `Retry`，经 `HTTPAdapter(max_retries=...)` 注入 requests）管连接错误与指定状态码，配 `backoff_factor` 做指数退避，尊重 `Retry-After`；业务层重试（tenacity）按异常类型与返回结果决定重试，适合包在「调一次 LLM 抽取」这类业务函数上。原文给出的退避数字是 `Retry(total=3, backoff_factor=0.5, status_forcelist=[429, 500, 502, 503, 504])`；原文小技巧节补了一句：backoff_factor=0.5 时三次重试的退避分别是 0 秒、1 秒、2 秒，第一次失败后立刻重试一次——指望“退避”挡住第一次冲击是错觉，它挡的是第二次、第三次。

### 流式读取为什么能分清“模型在想”还是“连接已死”

原文 §7 给出的机制是：`stream=True`（httpx 用 `client.stream(...)`）逐行读 NDJSON 增量，边收边落进度；卡住时能判断是模型在想还是连接已死，而不是等大响应干瞪眼。非流式调用在等大响应期间没有任何中间信号：读超时一到就断，断的时候不知道模型是还没开始想、正在想、还是连接已经断了；流式调用每收到一行增量就证明连接还活着、模型还在输出，进度一直在推进，超时只在“超过一段时间一行都没收到”时才触发。原文 §7 的落点把这条和重试边界连在一起：生成中途断流不要盲目整单重试，会重复生成、还可能得到另一份结果；引擎的口径是整章抽取一个事务，失败就整章回滚重抽，重试发生在章节级而不是请求级。流式调用还有一条配套约束：响应用完必须关，否则连接回不了池（原文小技巧节）——`with session.get(url, stream=True) as r:` 走完就关，不读完、不关闭，连接一直占着池位，池满后新请求就卡在“等连接池空位”上。

## 实战案例

### 案例一：日志打两遍，novel logger 挂了 handler 又让它冒泡到 root

- **现象**：逐章抽取跑几百章时，每条 INFO 日志在控制台出现两次，engine.log 里也出现两次；日志量翻倍，排障时同一条记录看两遍。
- **排查**：重复输出的经典根因就是自己挂了 handler 又让它冒泡到 root，打两遍（原文 §2）。对照本篇 dictConfig：`novel` logger 挂了 console 与 file 两个 handler，又显式设了 `propagate: False`；一旦把 `propagate: False` 漏掉，子 logger 的日志先走自己挂的两个 handler 输出一次，再冒泡到 root handler 又输出一次。根子在传播是默认开的：不显式关掉，每级 handler 都会再输出一次，挂得越多、重复越多。
- **结论**：统一入口一处 dictConfig，`novel` 这类业务 logger 显式设 `propagate: False`（原文 §8 与 §2）。库代码只打日志、不设级别不加 handler（顶多挂 NullHandler），级别与输出去向由应用入口统一决定（原文 §2）。`basicConfig` 在 import 靠后时“不生效”：它只在 root 未配置时起作用，各处乱调必然互相覆盖，统一入口一处 dictConfig（原文 §8）。

### 案例二：httpx 默认 5 秒超时，把正常的大章生成误杀了

- **现象**：批量抽取换 httpx 后，大章生成总在 5 秒左右断掉，报错超时；同一章用 requests 调却能正常跑完。
- **排查**：httpx 的默认超时只有 5 秒，别把它当 requests 用：`httpx.Client()` 不传 timeout 时默认 `Timeout(5.0)`，requests 不传 timeout 则是无限等（原文小技巧节）。对照原文 §7：调本地 Ollama 时生成是慢工，首 token 延迟加长输出，read 超时 60 秒会把正常的大章生成误杀，应按「连接 3 秒、读取数分钟」设。5 秒的默认超时连 60 秒的误杀线都没到，大章还没生成完就被断了。根子在两个库的默认超时语义完全不同：requests 默认无限等，httpx 默认 5 秒就断，迁移时不显式传超时等于悄悄改了超时策略。
- **结论**：用 httpx 一定要显式传 `timeout=httpx.Timeout(connect=3, read=300, write=10, pool=5)`，别指望默认配置对本地大模型友好（原文小技巧节）。或直接走流式（`stream=True` 逐行读 NDJSON），让超时只防真卡死（原文 §7）。失败打 ERROR 带状态码与响应前 200 字；httpx 自己的 logger 压到 WARNING，否则每次请求两行流水账刷屏（原文 §7）。

## 小技巧与冷知识

【小技巧】`logger.exception` 只能写在 except 块里：在 except 外调用它，打出来的堆栈是 `NoneType: None`，等于白打一行 ERROR。想在非异常处记堆栈用 `logger.error("...", exc_info=True)` 同样是错的——没有活动异常时它也只会打出 `NoneType: None`；没有异常就别带堆栈，堆栈只在 except 块里才有意义。

【小技巧】昂贵的日志参数先判级别再拼：`logger.debug("result: %s", expensive_dump(obj))` 里 `expensive_dump` 在 DEBUG 被关掉时照样先执行，惰性参数只省拼接、不省求值。正确写法是先 `if logger.isEnabledFor(logging.DEBUG): logger.debug(...)`，或者把格式化包进一个只在需要时才调用的 lambda/对象里；日志越详细的地方，越要先问一句"这个级别开着吗"。

【小技巧】`extra` 字典的 key 别和 LogRecord 的内置属性撞名：`logger.info("...", extra={"request_id": rid})` 是给日志加上下文的标准做法，但 key 一旦叫 `message`、`levelname`、`asctime` 这类保留名，Formatter 取值时会直接抛 KeyError。加上下文统一用 `request_id`、`chapter_id` 这种业务名前缀，或者用 LoggerAdapter 把上下文绑在 logger 上，别每次调用都传 extra。

【小技巧】多进程写同一个日志文件别共用 FileHandler：几个进程同时往一个文件里写，行与行会互相插队、甚至截断。标准做法是主进程里挂一个 QueueHandler + QueueListener 收日志，子进程只往队列里发；或者每个进程写自己的文件，事后按时间合并。文件锁、"反正量小不会撞"都是赌运气，队列才是 logging 官方给的答案。

【小技巧】httpx 的默认超时只有 5 秒，别把它当 requests 用：`httpx.Client()` 不传 timeout 时默认 `Timeout(5.0)`，requests 不传 timeout 则是无限等。原文说 Ollama 生成要"读取数分钟"，直接拿默认 httpx 调 Ollama，5 秒就断；用 httpx 一定要显式传 `timeout=httpx.Timeout(connect=3, read=300, write=10, pool=5)`，别指望默认配置对本地大模型友好。

【小技巧】调 `.json()` 之前先 `raise_for_status()`：服务端返回 500 配一段 HTML 错误页时，`resp.json()` 抛的是 JSONDecodeError，真正的 500 状态码和错误体全被这层异常盖住了。先 `raise_for_status()` 让 HTTPException 带状态码出来，再解析业务体；响应解析失败时日志里记状态码 + 响应前 200 字，比只记 JSONDecodeError 有用得多。

【小技巧】`stream=True` 的响应用完必须关，否则连接回不了池：requests 里 `with session.get(url, stream=True) as r:` 走完就关，不读完、不关闭，连接一直占着池位，池满后新请求就卡在"等连接池空位"上。httpx 的 `client.stream(...)` 同理，必须进上下文管理器；"读一半就走"是流式调用最常见的连接泄漏。

【小技巧】Client/Session 用完要关，别每次请求新建：`httpx.Client()` 和 `requests.Session()` 都持有连接池，不 close 会触发"未关闭连接池"的 ResourceWarning，长期跑的批处理还会慢慢漏连接。正确姿势是整个流程复用一个实例、用完 `client.close()`，或用 `with httpx.Client() as client:` / `async with httpx.AsyncClient() as client:` 让上下文管理器收尾。

【小技巧】urllib3 的第一次重试是不等待的：`backoff_factor=0.5` 时三次重试的退避分别是 0 秒、1 秒、2 秒，第一次失败后立刻重试一次。指望"退避"挡住第一次冲击是错觉，它挡的是第二次、第三次；想让第一次也等一等，只能把 backoff_factor 设大，或者用 tenacity 的 `wait_exponential` 从第一次就按指数等。

【小技巧】tenacity 默认最后抛的是 RetryError 而不是原异常：`@retry` 装饰的函数重试耗尽后，默认抛出 `tenacity.RetryError`，原异常被包在里面，调用方 `except ConnectionError` 抓不到。想让原异常透出来，加 `reraise=True`；否则上层要按 RetryError 写分支，错误处理逻辑会被重试装饰器悄悄改掉。

【小技巧】给 httpx 写单测别起真实服务，用 MockTransport：`httpx.MockTransport(handler)` 可以在内存里按请求返回假响应，断言发出去的 URL、请求头、请求体全都能在 handler 里拿到。调 Ollama 的抽取函数单测时，用 MockTransport 返回一段 NDJSON 假流，比"测试前先起 Ollama、测完再关"快得多，也不会让单测依赖本地服务。

【小技巧】Session 上挂多个 mount 时按最长前缀匹配：`session.mount("https://api.example.com", adapter_a)` 和 `session.mount("https://", adapter_b)` 同时存在时，发往 api.example.com 的请求走 adapter_a 而不是后挂上的那个。调 Ollama 本地地址想单独配超时/重试时，给 `http://127.0.0.1:11434` 单独 mount 一个 adapter，比全局改参数精准得多。

【小技巧】日志里不要打完整请求体，截断时要截在"结构完整"处：原文说响应截前 200 字，落到实处还有一层——截断点别卡在 JSON 中间，`resp.text[:200]` 经常截出半个字段，排障时反而误导。记状态码 + 响应头里的 content-type + 前 200 字，同时注明"已截断"，调用方看到截断标记就知道后面还有，别把半截 JSON 当完整数据去猜。

【冷知识】Python 的 logging 模块来自 PEP 282，由 Vinay Sajip 在 Python 2.3（2003 年）引入：它借鉴了 Java 的 log4j（logger 层级、handler、formatter 几乎一一对应），所以原文说 logging 与 Logback"概念一一对应"不是巧合，是同一套设计的两个实现。`logging.getLogger(__name__)` 按模块名分层的做法，也是从 Java 的包名分层直接搬来的。

【冷知识】没有任何 handler 时，WARNING 以上的日志不会悄悄丢掉：Python 3.2 起 logging 有个"最后手段"处理器 `logging.lastResort`，它会把 WARNING 及以上级别的日志打到 stderr。脚本里忘了配日志、又看到控制台冒出 WARNING，就是它在兜底；它只负责"别静默丢日志"，格式、落盘还得自己配。

【冷知识】requests 的作者 Kenneth Reitz 给它的定位是"HTTP for Humans"，名字里的 requests 本身就是对 urllib 难用 API 的回应：它把"发一个 GET 请求"从 urllib 的七八行样板压缩到一行。httpx 的作者 Tom Christie 也写过 Django REST framework，httpx 的 API 刻意和 requests 长得像，就是为了让 requests 用户零成本迁移到异步。

【冷知识】`logging.CRITICAL` 和 `logging.FATAL` 是同一个级别（都是 50）：FATAL 是从 log4j 时代留下的别名，Python 文档推荐用 CRITICAL。级别名字里还有一处历史遗留：`logging.warn` 是 `warning` 的旧别名，Python 3.3 起已废弃，见到老代码里的 `logger.warn` 就知道那是 3.3 之前的写法。

## 面试与自测

1. 同一条日志为什么会打两遍？库代码在日志配置上应该遵守什么原则？
2. requests 的 `timeout=(3, 60)` 两段各指什么？调本地 Ollama 时为什么 read 超时 60 秒会把正常的大章生成误杀？
3. 重试的边界是什么？为什么 Ollama 生成中途断流不能盲目整单重试？

### 参考答案

1. 重复输出的经典根因是自己挂了 handler 又让它冒泡到父级：日志沿层级向父级传播（propagate 默认开），子 logger 的日志会冒泡到 root handler，于是打两遍。本篇 dictConfig 里 `novel` logger 就显式设了 `propagate: False`。库代码只打日志、不设级别不加 handler（顶多挂 NullHandler），级别与输出去向由应用入口统一决定。
2. requests 用元组拆两段：`timeout=(连接超时, 读取超时)`，如 `timeout=(3, 60)`。requests 默认超时是无穷大，服务端 hang 住客户端就永久挂起。调 Ollama 时生成是慢工（首 token 延迟 + 长输出），read 超时 60 秒会把正常的大章生成误杀，应按「连接 3 秒、读取数分钟」设，或直接走流式（`stream=True` 逐行读 NDJSON），让超时只防真卡死。
3. 只重试连接失败、超时与 5xx/429；4xx 是请求本身错了，重试纯属浪费。POST 等非幂等请求慎重试——服务端可能已处理成功只是响应丢了，要重试先确认有幂等键或操作可重入。Ollama 生成中途断流不要盲目整单重试，会重复生成、还可能得到另一份结果；引擎的口径是整章抽取一个事务，失败就整章回滚重抽，重试发生在章节级而不是请求级。

## 9. 关联知识

### 前置知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引，本篇的日志与 HTTP 客户端都是 Python 标准库与第三方库的用法，先有语言基础再读本篇的配置与选型。
- [Logback 与日志规范](工程实践/Logback与日志规范.md)：Java 侧日志体系，原文 §1 点名 logging 的 logger 层级、handler、formatter 与 Logback 的 logger/appender/layout 概念一一对应，概念对位先读它。
- [asyncio](开发语言/Python/asyncio.md)：httpx 异步客户端的并发底座，原文 §4 点名抽取要并发调 Ollama 时走 httpx 的 `AsyncClient`，与 asyncio 篇的并发模型接上。
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)：原文 §7 点名引擎的口径是整章抽取一个事务，失败就整章回滚重抽，重试发生在章节级而不是请求级，事务边界先读它。
- [本地部署与 Ollama](AI与大模型/本地部署与Ollama.md)：被调用方的部署与参数，原文 §7 的落点全是调本地 Ollama 的实战姿势，调用方的超时与流式要对照被调用方的行为来设。

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [Logback 与日志规范](工程实践/Logback与日志规范.md)：Java 侧日志体系，概念对位
- [asyncio](开发语言/Python/asyncio.md)：httpx 异步客户端的并发底座
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)：整章一个事务，失败回滚重抽
- [本地部署与 Ollama](AI与大模型/本地部署与Ollama.md)：被调用方的部署与参数
