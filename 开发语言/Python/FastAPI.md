---
知识点: FastAPI
层级: 基础
适用版本: FastAPI（基于 Starlette + pydantic v2）
更新日期: 2026-10-31
关联:
  - Python
  - pydantic
  - asyncio
标签:
  - Python
  - 实践
---

# FastAPI

## 1. 核心概念

FastAPI 是基于 Starlette（ASGI 框架）+ pydantic 的现代 Web 框架，核心卖点是**用类型注解同时驱动三件事**：请求参数解析、数据校验、OpenAPI 文档生成。函数签名写清楚类型，框架就知道怎么收参、怎么验、文档怎么写，不再手写一套、文档另维护一套。

与 Flask 的分工判断：一次性脚本、内部小工具 Flask 够用；要长期维护、要给前端/Agent 当接口契约、要异步并发的服务，FastAPI 更合适。小说引擎的落点就是后者——把章节分析、RAG 检索、矛盾检测包成本地 HTTP 服务（`127.0.0.1` 监听），UI 或其他 Agent 进程通过接口调用，而不是全塞在一个进程里。

启动与运行：开发用 `fastapi dev` / `uvicorn app:app --reload`，本质都是 uvicorn 这个 ASGI 服务器在跑；版本与命令细节以官方文档为准。

## 2. 路由与参数：注解即契约

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel

app = FastAPI()

class ChapterIn(BaseModel):
    project_id: int
    number: int
    content: str

@app.get("/chapters/{chapter_id}")
def get_chapter(chapter_id: int, project_id: int = Query(...)):
    ...

@app.post("/chapters/analyze", status_code=202)
def analyze(payload: ChapterIn):
    ...
```

- 路径参数、查询参数、请求体由**参数位置和类型自动区分**：单值类型是查参，`BaseModel` 是请求体，不用手写解析。
- 校验失败自动返回 422 + 结构化错误清单（就是 pydantic 的 `errors()` 格式），调用方能直接读。请求模型上加 `model_config = ConfigDict(extra="forbid")`，多余字段直接拒，和 LLM 结构化输出同一套纪律。
- 响应侧用 `response_model=` 声明出参模型：框架按它过滤字段（密码、内部状态不会顺手漏出去），文档也按出参画。

## 3. 依赖注入：Depends

FastAPI 的依赖注入是它区别于 Flask 的关键机制：把"每请求都要做的事"——拿 DB Session、读配置、校验令牌——声明成依赖，框架按请求自动执行并注入：

```python
from fastapi import Depends
from sqlalchemy.orm import Session

def get_session():
    with SessionLocal() as session:   # 见 SQLAlchemy 篇
        yield session                 # yield 型依赖：请求结束自动收尾

@app.get("/entities")
def list_entities(session: Session = Depends(get_session)):
    ...
```

- 依赖可嵌套（依赖里再依赖）、可复用、可用 `dependency_overrides` 在测试里整体替换成假实现——这点对 pytest 非常友好，不用打补丁。
- 类比 Spring 的 `@Autowired` + 请求作用域 Bean：`yield` 前是"构造 + 注入"，yield 后是 `@PreDestroy`。从 Java 过来按这个模型理解最快。

## 4. pydantic 衔接

FastAPI 的入参/出参模型就是 pydantic `BaseModel`，与 [pydantic 篇](开发语言/Python/pydantic.md) 是同一套：

- 请求模型管**进系统的数据形状**，响应模型管**出系统的形状**，两者分开定义，别用一个模型双向共用（落库字段、内部字段迟早要分家）。
- 引擎的分析接口可以复用章节分析的 pydantic 模型当请求体——但注意分工：FastAPI 层校验的是"调用方传的形状"，LLM 输出的校验在服务层内部，两个边界不要合并成一次校验。
- 配置走 `pydantic-settings` 的 `BaseSettings` 并缓存成单例依赖（`@lru_cache`），别每个请求重新读环境变量。

## 5. 异步端点与阻塞坑（最重要的一节）

路由函数两种写法，选错就丢并发：

- `async def` 端点跑在事件循环上，**里面绝不能有阻塞调用**：同步 SQLAlchemy Session、同步 `requests`、本地 Ollama 的同步 HTTP 客户端、长时间 CPU 计算（如大文本分块）——任何一个都会把整个事件循环冻住，所有请求一起卡。
- `def`（同步）端点框架会自动丢到线程池跑，阻塞只占一个线程，不冻全局。**引擎这类以同步 SQLAlchemy + 同步 Ollama 调用为主的服务，端点默认写同步 `def` 是更稳的选择**，别为了"用了 FastAPI 就该全异步"硬上 async。
- 真要 async：数据库换异步驱动（aiosqlite + AsyncSession）、HTTP 换 `httpx.AsyncClient`，整条链路全异步才有意义；混一半是最坏状态——async 端点里藏着同步调用，压测才暴露。
- CPU 密集且耗时的活（整章分析、embedding 批处理）本质不该挂在请求线程里等：接口收下任务返回 202 + 任务 id，后台执行，调用方轮询或回调。这与 [asyncio 篇](开发语言/Python/asyncio.md) 的任务编排是同一件事，HTTP 层只是入口。

## 6. 给 Agent / 本地工具做服务的落点

- **本地优先、只绑回环**：引擎服务监听 `127.0.0.1` 即可，不暴露局域网；真要跨机调用再加令牌校验依赖，不要默认裸奔 `0.0.0.0`。
- **接口即工具**：给 Agent 调的接口粒度按"一个完整动作"设计（提交章节分析、查实体当前状态、跑一次矛盾检测），返回结构化 JSON（pydantic 出参模型保证形状稳定），Agent 侧就能把它当 Tool Calling 的一个工具直接对接——参数 schema 从 OpenAPI 文档自动生成，不用手写第二份。
- **长任务给进度**：分析类接口返回任务 id 后，提供状态查询端点（排队/运行/完成/失败 + 错误信息），失败原因结构化返回，别让调用方只能靠超时猜。
- **文档即契约**：`/docs` 的交互文档由注解自动生成，改了模型就同步变；拿它当与调用方的契约基准，不再另写接口文档（与知识库里 OpenAPI 篇的思路一致）。

## 7. 常见误区

- **async 端点里写同步数据库/HTTP 调用**：见第 5 节，本框架第一大坑。
- **在路由函数里写业务逻辑**：路由只做收参、调 Service、返回；业务仍归 ChapterService 这类服务层（与引擎现有分层一致），否则接口一多逻辑就散。
- **全局可变状态存请求间数据**：模块级 dict 存"当前项目"之类状态，多 worker/多请求立刻串数据；请求级状态走依赖注入或显式参数。
- **异常裸抛给框架默认 500**：业务异常（如章节不存在）定义异常处理器返回明确状态码与结构化信息，调用方是程序不是人，错误契约和成功契约一样重要。

## 小技巧与冷知识

【小技巧】依赖的返回值用不上时，别写进参数里，挂到装饰器上：`@app.get("/x", dependencies=[Depends(verify_token)])` 只执行鉴权、不注入返回值，路由函数签名保持干净。参数里的 `Depends` 是“要结果”，装饰器里的 `dependencies` 是“只要执行”，鉴权、限流这类只求通过的依赖放后者，读函数签名的人一眼就知道哪些是输入、哪些是门禁。

【小技巧】同一个依赖在一个请求里被多处引用时，默认只执行一次：FastAPI 的 `Depends` 默认 `use_cache=True`，同一请求内多次依赖同一个可调用对象，框架只算一次、结果复用。DB Session 依赖被三个子依赖同时引用时，不会开三个连接；想让每次都重新算（比如每次都要读一次最新配置），显式写 `Depends(fn, use_cache=False)`，别靠“多写几次就多算几次”猜行为。

【小技巧】启动/关闭逻辑用 `lifespan` 上下文管理器，别再用 `@app.on_event("startup")`：`lifespan` 是 Starlette 推荐的写法，`yield` 前做初始化（建连接池、预热模型）、`yield` 后做收尾，`on_event` 已被标记废弃。启动期要做的事（比如小说引擎预热 Ollama 连接）放 `yield` 前，失败了服务直接起不来，比请求打进来才发现没准备好要强得多。

【小技巧】“响应完再慢慢做”的活交给 `BackgroundTasks`，别在路由里等它：发通知、写审计日志这类不需要调用方等的收尾，路由参数里声明 `background_tasks: BackgroundTasks`，`add_task` 后直接返回，任务在响应发出后才跑。注意它只是“响应后执行”，不是任务队列——进程一重启就丢，重要任务还是要走真正的队列（见第 5 节的 202 + 任务 id 思路）。

【小技巧】出参模型配 `response_model_exclude_unset=True`，只返回真正被赋值的字段：模型里大量字段是 `Optional` 默认 `None` 时，响应里会塞满一堆 `null`；加这个参数后没被显式设置的字段直接不出现，响应变小、调用方也不用猜“这个 null 是没值还是没算”。只对出参模型用，别把它和入参模型混在一起。

【小技巧】路由多了就用 `APIRouter` 按模块拆，别把所有 `@app.get` 堆在一个文件里：`router = APIRouter(prefix="/chapters", tags=["chapters"])`，各模块各建一个 router，最后 `app.include_router()` 汇总。`prefix` 统一收口径，`tags` 让 /docs 里自动按模块分组；接口一多，单文件路由是第一个撑不住的地方。

【小技巧】测试接口别起真实服务器，用 `TestClient`：`from fastapi.testclient import TestClient; client = TestClient(app)`，直接在进程内调接口，不用等 uvicorn 启动、不用占端口。配合 `dependency_overrides` 把 DB 依赖换成内存实现，pytest 里跑完整条请求链路比起服务再 curl 快得多；`TestClient` 底层是 httpx，行为和真实请求一致。

【小技巧】要改响应头或 Cookie 时，在路由参数里声明 `Response`：`def f(response: Response)`，函数里 `response.headers["X-Request-Id"] = ...`，不用为了改个头就把返回值改成 `JSONResponse`。返回值继续写业务模型，框架按 `response_model` 序列化；`Response` 参数只管“顺带改点响应元数据”，两者分工别混。

【小技巧】查询参数的默认值别写裸值，写 `Query(...)` 才能加约束：`q: str = Query(default=None, max_length=50)` 既能给默认值又能限长度，裸写 `q: str = None` 就没了校验入口。别名也一样，`Query(alias="pageSize")` 让接口接受驼峰参数名、函数里仍用下划线变量——外部命名和内部命名分开，是 Query 的本职工作。

【小技巧】生产环境关掉交互文档：`FastAPI(docs_url=None, redoc_url=None)`，/docs 和 /redoc 直接 404。开发环境文档是契约，生产环境它是攻击面——接口形状、参数名、错误示例全暴露在外网等于送了一份说明书；本地开发照常用，部署时用环境变量控制开关，别让文档跟着服务一起上线。

【小技巧】业务错误别 `return {"error": ...}`，抛 `HTTPException`：`raise HTTPException(status_code=404, detail="章节不存在")`，状态码、结构化错误体由框架统一出，调用方按状态码分支。路由里 `return` 字典永远是 200，调用方分不清“成功”和“失败但给了个 JSON”；错误契约和成功契约一样，都要靠状态码说话。

【冷知识】FastAPI 是 Sebastián Ramírez 在 2018 年 12 月发布的：它站在 Starlette（ASGI 层）和 Pydantic（校验层）肩上，靠 Python 3.6+ 的类型注解把“参数解析、校验、文档”三件事一次做完。框架名里的 Fast 既指开发快，也指它跑在 ASGI + Uvicorn 上性能不输 Go 系框架的基准测试成绩。

【冷知识】/docs 是 Swagger UI，/redoc 是 ReDoc，两个都是同一份 OpenAPI 文档的不同皮肤：FastAPI 启动后自动挂出两个文档地址，/docs 侧重交互试调，/redoc 侧重长文档阅读。改了模型或路由不用重新生成文档，刷新页面就是最新的——文档不是写出来的，是类型注解算出来的。

【冷知识】FastAPI 默认用标准库 `json` 序列化响应，想换更快的序列化器不用改业务代码：`from fastapi.responses import ORJSONResponse`，建 app 时 `default_response_class=ORJSONResponse`，所有 JSON 响应改走 orjson。响应体大的接口（比如一次返回整章分析结果）收益最明显；换序列化器是 app 级配置，不是每个路由的活。

## 面试与自测

1. async def 端点里写同步 SQLAlchemy 或同步 requests 会怎样？小说引擎这类服务端点默认写什么？
2. Depends 的 yield 型依赖在请求生命周期里起什么作用？测试时怎么替换它？
3. 请求模型和响应模型为什么要分开定义？response_model 还起什么作用？

### 参考答案

1. 会把整个事件循环冻住，所有请求一起卡——因为 async def 端点跑在事件循环上，里面绝不能有阻塞调用（同步 SQLAlchemy Session、同步 requests、同步 Ollama 客户端、长时间 CPU 计算都不行）。def（同步）端点框架会自动丢到线程池跑，阻塞只占一个线程，不冻全局；引擎这类以同步 SQLAlchemy + 同步 Ollama 调用为主的服务，端点默认写同步 def 是更稳的选择，别为了“用了 FastAPI 就该全异步”硬上 async。
2. yield 前是“构造 + 注入”，yield 后是请求结束自动收尾（类比 Spring 的 @PreDestroy），例如每请求拿 DB Session、用完自动关闭。依赖可嵌套、可复用，测试时用 dependency_overrides 整体替换成假实现，不用打补丁。
3. 因为落库字段、内部字段迟早要分家，别用一个模型双向共用；而且分工不同：请求模型管进系统的数据形状，响应模型管出系统的形状。response_model 还会按出参模型过滤字段（密码、内部状态不会顺手漏出去），文档也按出参画。

## 8. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [pydantic 数据校验](开发语言/Python/pydantic.md)：入参/出参模型与校验机制的底座
- [asyncio](开发语言/Python/asyncio.md)：事件循环、阻塞与任务编排，理解第 5 节的前置
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)：依赖注入里每请求 Session 的来源

## 9. 一句话总结

FastAPI = 注解即契约的 ASGI 框架：pydantic 管形状、Depends 管每请求的资源、同步/异步选对了它就稳，选错了它就卡。
