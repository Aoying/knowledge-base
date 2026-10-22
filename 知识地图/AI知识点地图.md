# AI 知识点地图

## 知识点清单

- [[大模型基础]]：Transformer、Token、Embedding、量化、上下文、注入安全
- [[Agent]]：LLM + Tools + Memory + 循环控制，ReAct 与工程要点
- [[Skills-索引]]：已收录 Skills 的总索引
- [[RAG-Skills]]：RAG 相关 Skills
- [[Agent-开发-Skills]]：Agent 开发相关 Skills
- [[开发工作流-Skills]]：开发工作流相关 Skills

- Agent 进阶：[[LangGraph]]、[[MCP]]、[[Tool-Calling]]
- RAG 进阶：[[RAG-评测与分块策略]]

## 学习路径

```text
[[大模型基础]]
  → [[Agent]]
  → [[RAG-Skills]]（检索增强）
  → [[Skills-索引]]（按需取用具体 Skill）
```

## 前置依赖

- [[Python]]：本地 Ollama 与 Agent 工程的主要语言
- [[Elasticsearch]] / [[Redis]]：向量与语义检索的存储对照

## 关联方向

- 本地模型策略：14B 常驻（抽取、JSON、RAG）+ 约 30B 按需（复杂推演、终裁）
- 应用项目：通用小说创作引擎（RAG + 长期记忆 + 沙盒推演）

## 知识缺口

- RAG 评测实测数据待小说引擎原型验证后回填 [[RAG-评测与分块策略]]
- Agent 记忆与多 Agent 协作尚无独立篇（按需）
