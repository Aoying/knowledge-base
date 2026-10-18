# Agent

> 每晚 20:30 自动补充完善。

## 核心概念

一个 Agent = **LLM（推理）+ Tools（行动）+ Memory（记忆）+ 循环控制**。

最经典的循环是 **ReAct**：Thought（思考下一步）→ Action（调工具）→ Observation（看结果），循环直到任务完成。绝大多数 Agent 框架都是这个循环的变体。

## 四个关键部件

### 1. 推理（LLM）
负责做决策：下一步调哪个工具、传什么参数。本地可用 Ollama 跑支持 tool calling 的模型（如 qwen3 系列），云端按需用大模型做复杂裁决。**二层架构**是实用套路：小模型常驻处理高频常规任务，大模型按需处理复杂推理——和你"14B 常驻 + 30B 按需"的规划一致。

### 2. 工具（Tools / Tool Calling）
- 每个工具给 LLM 的是一个 **function schema**：名称、一句话描述（写清楚触发条件）、参数定义。
- 工程要点：参数校验（schema 对不上就拒掉重问）、错误重试（工具报错把报错信息喂回 LLM 让它自己修参数）、最小权限（工具只能做它该做的事）。
- 危险操作（删库、发消息、花钱）必须走人工确认，不能让 Agent 自己决定。

### 3. 记忆（Memory）
- **短期**：当前对话的上下文窗口，放最近的 Thought/Action/Observation。
- **长期**：结构化存（SQLite/向量库）——实体、关系、事件、待办。你的小说引擎 Entity/Relation/Event/Timeline 就是长期记忆的一种实现。
- 经验：长期记忆的关键不是"存得多"，而是"检索得准"——检索策略（关键词 + 向量混合）比存储格式更重要。

### 4. 循环控制
- 什么时候停：任务完成 / 步数上限 / LLM 明确说做完了。三者缺一不可，否则 Agent 会空转烧 token。
- **Human-in-the-loop**：关键节点停下来问人，确认后再继续。你的"高风险操作先确认"就是这个思想。

## LangGraph（状态机式 Agent 框架）

核心只有四个概念，适合你这种有后端/流程引擎背景的人：

- **State**：流程中传递的状态对象（类似 Flowable 的流程变量）。
- **Node**：一个处理步骤（调 LLM、调工具、写库）。
- **Edge**：步骤之间的流转，支持条件分支（类似排他网关）。
- **Checkpointer**：状态持久化，支持断点续跑和"回到某一步重来"——这是 LangGraph 相对裸写循环最大的工程价值。

心智映射：Node≈服务任务节点，Edge≈顺序流/条件流，State≈流程变量，Checkpointer≈流程实例持久化。有 Flowable 经验学这个很快。

## MCP（Model Context Protocol）

一句话：**Agent 调外部能力的统一插头标准**。Server 提供 tools/resources/prompts，Client（你的 Agent）按标准去发现和调用。不用再为每个 API 手写一套对接胶水。

## Hermes Agent

你在研究的方向。Hermes 系模型以 tool calling / function calling 见长，适合做本地 Agent 的推理核心。NVIDIA 官方把 nemotron 系列接入了 Hermes、OpenClaw 生态——本地 Agent 这条线可以持续跟。

## 落到你的项目

- 小说引擎的 EventEngine → 可以 Agent 化：把"抽取实体/关系/事件"做成带校验和重试的 Agent 循环，而不是一次性 LLM 调用。
- 沙盒推演 → 本质是"分支 + 并行跑多个 Agent + 对比结果"，LangGraph 的分支能力正好对上。
- 冲突检测 → 二层架构：小模型常驻扫全量章节，大模型按需做最终裁决。


## Agent 设计模式（四种常用）

1. **ReAct**：Reason + Act 循环，"想一步 → 调一个工具 → 看结果 → 再想"。最通用，LangChain 默认就是这个。
2. **Plan-and-Execute**：先让 LLM 出完整计划，再一步步执行，执行偏离时重计划。适合长流程（如你的小说多章生成）。
3. **Reflection**：做完自己评审一遍，"这段剧情有没有矛盾？"，有问题就改。对应你的 ChapterAnalysisService → 修正循环。
4. **Multi-Agent**：分工，一专多能不如多专：抽取 Agent、矛盾检测 Agent、文风 Agent 各干各的，最后汇总。LangGraph 的 supervisor 模式就是干这个的。

- 小说引擎现在用的是"单 Agent + 结构化输出"，下一步可以试 ReAct（抽取时主动查时间线）+ Reflection（生成后自检矛盾）。

## 来源

- 概念部分为通用知识整理（ReAct / Tool Calling / MCP / LangGraph 官方文档）。
- 与本地模型相关的内容以 Ollama 官方库实测为准。
