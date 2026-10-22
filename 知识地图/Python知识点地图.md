# Python 知识点地图

> Python 位于 `开发语言/Python/`，按知识点拆分：`Python.md` 为语言基础与总索引。

## 知识点清单

- [[Python]]：语言基础、环境与依赖、标准库速查、sqlite3、Ollama 调用、常见坑
- [[asyncio]]：并发模型（小说引擎批量抽取章节时用）
- [[pydantic]]：数据校验（LLM 输出 JSON 落库前）
- [[dataclass与typing]]：dataclass + typing，告别 dict 满天飞
- [[pytest]]：测试实战
- 开发语言同层：[[JavaScript]]、[[Lua]]、[[正则表达式]]

## 学习路径

```text
[[Python]]
  → [[dataclass与typing]] → [[pydantic]]
  → [[asyncio]]
  → [[pytest]]
  → [[Agent]] / [[RAG-Skills]]（Python + LLM 应用方向）
```

## 前置依赖

- [[数据结构与算法]]：列表、字典、集合的复杂度基础
- [[编码-字符集-内存-文件系统]]：字符串与编码问题的前置

## 关联方向

- 目标方向：Python + LLM + RAG + LangGraph + Agent 开发（转型路线）
- 语言对照：[[Java]]（后端经验迁移）、[[JavaScript]]（前端脚本）

## 知识缺口

- LangGraph、MCP 待随学习进度建篇（MCP 实战目前在 [[Agent]] 内）
