# AI 知识点地图
Up: [[总览]]

## 知识点清单

- [[大模型基础]]：Transformer、Token、Embedding、量化、上下文、注入安全
- [[Agent]]：LLM + Tools + Memory + 循环控制，ReAct 与工程要点
- [[Skills-索引]]：已收录 Skills 的总索引
- [[RAG-Skills]]：RAG 相关 Skills
- [[Agent-开发-Skills]]：Agent 开发相关 Skills
- [[开发工作流-Skills]]：开发工作流相关 Skills

- Agent 进阶：[[LangGraph]]、[[MCP]]、[[Tool-Calling]]
- RAG 进阶：[[RAG-评测与分块策略]]
- 基础与工程：[[Embedding与向量数据库]]、[[本地部署与Ollama]]、[[Prompt-工程]]、[[结构化输出与JSON约束]]、[[机器学习基础]]、[[深度学习基础]]
- Agent 深化：[[Agent-记忆系统]]、[[多Agent协作]]
- 上下文与安全：[[上下文工程]]、[[提示注入与Agent安全]]
- 检索进阶：[[混合检索与Rerank]]（与 [[RAG-评测与分块策略]] 配套）
- 子目录：Agent 6 篇在 `AI与大模型/Agent/`（Agent、Agent-记忆系统、多Agent协作、LangGraph、MCP、Tool-Calling），RAG 5 篇在 `AI与大模型/RAG/`（Embedding与向量数据库、RAG-评测与分块策略、混合检索与Rerank、上下文工程、向量检索深水）
- [[LLM生产化与推理优化]]（模型路由、语义缓存、降级与可观测）
- 评测与推理：[[模型评测方法]]、[[推理框架对比]]、[[微调入门-LoRA]]
- 训练与对齐：[[强化学习基础概念]]（RL 要素、RLHF/DPO/GRPO 区分）

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

- ~~上下文工程 / 混合检索与Rerank / 提示注入与Agent安全~~（2026-10-23 已补 [[上下文工程]]、[[混合检索与Rerank]]、[[提示注入与Agent安全]]，缺口移除）

- RAG 评测实测数据待小说引擎原型验证后回填 [[RAG-评测与分块策略]]
- ~~Agent 记忆与多 Agent 协作尚无独立篇~~（2026-10-23 已补 [[Agent-记忆系统]]、[[多Agent协作]]，缺口移除）
