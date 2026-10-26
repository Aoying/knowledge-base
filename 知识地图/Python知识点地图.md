# Python 知识点地图

> Python 位于 `开发语言/Python/`，按知识点拆分：`Python.md` 为语言基础与总索引。

## 知识点清单

- [[Python]]：语言基础、环境与依赖、标准库速查、sqlite3、Ollama 调用、常见坑
- [[asyncio]]：并发模型（小说引擎批量抽取章节时用）
- [[pydantic]]：数据校验（LLM 输出 JSON 落库前）
- [[dataclass与typing]]：dataclass + typing，告别 dict 满天飞
- [[pytest]]：测试实战
- [[SQLAlchemy]]：小说引擎的 ORM 与数据层
- [[Python-多进程与GIL]]：GIL 与线程/进程/协程选型
- [[Python-日志与HTTP客户端]]：logging 体系与调 Ollama 的 HTTP 姿势
- [[Python-面向对象基础]]：类、继承、多态、魔术方法
- [[Python-线程基础]]：threading 实操与 GIL 边界
- [[FastAPI]]：给 Agent 与本地工具做服务
- [[Python-打包与环境管理]]：venv、pyproject、uv
- [[Python性能优化]]：定位、常见慢点、优化手段、过早优化误区
- 开发语言同层：[[JavaScript]]、[[Lua]]、[[正则表达式]]、[[正则表达式-常用样例]]

## 学习路径

```text
[[Python]]
  → [[dataclass与typing]] → [[pydantic]]
  → [[asyncio]]
  → [[pytest]]
  → [[SQLAlchemy]] → [[FastAPI]]
  → [[Agent]] / [[RAG-Skills]]（Python + LLM 应用方向）
```

## 前置依赖

- [[数据结构与算法]]：列表、字典、集合的复杂度基础
- [[编码-字符集-内存-文件系统]]：字符串与编码问题的前置

## 关联方向

- 目标方向：Python + LLM + RAG + LangGraph + Agent 开发（转型路线）
- 语言对照：[[Java]]（后端经验迁移）、[[JavaScript]]（前端脚本）

## 知识缺口

- ~~Python 日志与 logging + HTTP 客户端与重试~~（2026-10-23 已补 [[Python-日志与HTTP客户端]]，缺口移除）

- 暂无登记缺口（装饰器、生成器已补篇；LangGraph、MCP 已在 AI 领域建篇）
