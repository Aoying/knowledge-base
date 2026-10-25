---
知识点: LangGraph
层级: 进阶
适用版本: LangGraph 1.x（Python）
更新日期: 2026-10-22
关联:
  - Agent
  - Tool-Calling
  - MCP
标签:
  - AI
  - Agent
  - 实践
---

# LangGraph

> 状态图式 Agent 编排框架：把 Agent 的循环控制从「一个 while 循环」升级成「可持久化、可分支、可回滚的状态机」。适合有流程引擎背景的人——心智模型和 Flowable 几乎一一对应。

## 1. 核心概念

LangGraph 用一张**有向图**描述 Agent 的执行流程，核心只有四个概念（源自 Agent 篇「LangGraph（状态机式 Agent 框架）」一节）：

- **State**：全图共享的状态对象，在节点之间传递（类似 Flowable 的流程变量）。
- **Node**：一个处理步骤——调 LLM、调工具、写库、做校验，都是节点（类似服务任务节点）。
- **Edge**：节点之间的流转，支持条件分支（类似顺序流 / 排他网关）。
- **Checkpointer**：状态持久化层，每一步执行后把 State 存下来，支持断点续跑和「回到某一步重来」——这是 LangGraph 相对裸写循环最大的工程价值（类似流程实例持久化）。

什么时候用它、什么时候不用：

- 流程里有**循环**（重试、自检、反思）、**分支**（按结果走不同路径）、**并行**（多个子任务同时跑）、**中途等人**（人工确认）时，用图比手写循环清晰。
- 只是一次「拼 prompt → 调模型 → 解析 JSON」的直线调用，不要上框架，普通函数就够。

## 2. 基础知识

最小可运行示例：一个带工具调用的 ReAct 图。

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

class State(TypedDict):
    # add_messages 是 reducer：节点返回新消息时追加而不是覆盖
    messages: Annotated[list, add_messages]
    retry_count: int

def call_model(state: State):
    # 这里换成你的模型调用（Ollama / OpenAI 兼容接口都行）
    response = model.invoke(state["messages"])
    return {"messages": [response]}

builder = StateGraph(State)
builder.add_node("agent", call_model)
builder.add_node("tools", ToolNode([query_timeline, check_contradiction]))
builder.add_edge(START, "agent")
# tools_condition：模型发起了 tool_calls 就去 tools 节点，否则结束
builder.add_conditional_edges("agent", tools_condition, {"tools": "tools", END: END})
builder.add_edge("tools", "agent")

conn = sqlite3.connect("agent_state.db", check_same_thread=False)
app = builder.compile(checkpointer=SqliteSaver(conn))

# thread_id 隔离一次任务的全部状态，相当于流程实例 ID
config = {"configurable": {"thread_id": "chapter-101-analysis"}}
result = app.invoke({"messages": [{"role": "user", "content": "分析第 101 章"}],
                     "retry_count": 0}, config)
```

要点：

- 节点是普通 Python 函数：**入参是 State，返回值是要更新的字段字典**，不要原地改 State。
- State 字段默认「覆盖」语义；要追加（如消息列表）必须用 `Annotated[..., reducer]` 声明合并方式，这是新手最常踩的坑。
- `compile()` 时不传 checkpointer 也能跑，但状态只在内存里，进程一退全丢——只要任务超过一次调用，就该挂 checkpointer。

## 3. 进阶知识

**条件边与循环**：路由函数只读 State、返回下一个节点名，不做实际工作。ReAct 的「调工具→回模型」就是一条指回上游的边构成的环。环必须有退出条件（步数、重试次数或明确的完成标志），并配合 `recursion_limit` 兜底：

```python
app.invoke(state, {**config, "recursion_limit": 25})  # 默认 25 步，超限直接抛错
```

**人机回路（interrupt）**：节点里调用 `interrupt()` 可以把图暂停，把待确认内容抛给外部，拿到回复后用 `Command(resume=...)` 从断点继续——状态已在 checkpointer 里，不用重跑前面的步骤。小说引擎的「章节生成后人工确认再落库」就适合这个模式：

```python
from langgraph.types import interrupt, Command

def publish_gate(state: State):
    decision = interrupt({"action": "发布本章", "draft": state["draft"][:200]})
    return {"approved": decision == "yes"}

# 外部恢复：
app.invoke(Command(resume="yes"), config)
```

**子图与多 Agent**：一张编译好的图可以作为另一张图的节点。Supervisor 模式就是主图负责分发、每个子图是一个专职 Agent（抽取、矛盾检测、文风检查），对应 Agent 篇里的 Multi-Agent 模式。子图可以共享父图 State，也可以定义自己的 State 再做映射——后者隔离更好，调试成本更高。

**流式输出**：`app.stream(..., stream_mode="updates")` 按节点产出增量结果，长任务（如多章批处理）可以用它做进度展示和早期失败发现。

## 4. 原理剖析

- **执行模型是超步（super-step）**：借鉴 Pregel/BSP 模型，一轮里所有「被激活」的节点并行执行，轮末统一把更新合并进 State 再进入下一轮。没有被边指向的节点不会跑；同一字段在同一轮被多个节点写、又没有 reducer，会直接报错而不是悄悄覆盖。
- **Channel 与 reducer**：State 的每个字段背后是一个 channel，节点返回值先写入 channel，再由 reducer 决定合并方式（覆盖、追加、自定义函数）。理解这一层，才能解释「为什么我返回了值却没生效/被追加了两份」类问题。
- **检查点存什么**：每个超步结束后存一份完整 State 快照 + 下一步计划，按 `thread_id` 组织成一条时间线。因此「回滚到第 N 步」本质是加载旧快照重新往前跑；`get_state_history()` 能列出全部历史检查点。
- **Checkpointer 实现**：内存版（`InMemorySaver`，仅调试）、SQLite 版（`SqliteSaver`，单机够用，和小说引擎同栈）、Postgres 版（多进程/生产）。三者接口一致，换实现不动图代码。

## 5. 实战应用

落到小说引擎（承接 Agent 篇「落到你的项目」）：

- **章节分析图**：节点 = 上下文组装（ChapterContextService）→ LLM 抽取 → JSON 校验 → 矛盾检测 → 落库。校验失败走条件边回到抽取节点重试（带 `retry_count`，上限 2 次），而不是整条流水线重跑。
- **沙盒推演**：同一初始 State 分出多个 `thread_id` 并行跑不同分支（苏雨存活/死亡/逃脱），最后用对比节点汇总三条时间线的检查点差异——分支和回滚都是 checkpointer 的原生能力，不用自己维护快照表。
- **终裁分层**：图里可以混用两个模型——常驻小模型节点做高频抽取，条件边只在「检测到疑似矛盾」时才路由到大模型节点做终裁，对应二层架构。
- 落库类节点必须**幂等**：检查点恢复会重跑当前步，写库前先按业务键查重（章节 ID + 版本），避免恢复一次多一份数据。

## 6. 疑难杂症

- **图跑到一半状态丢了**：多半是 invoke 时没传同一个 `thread_id`，或每次都新建 checkpointer 连接。先打印 `app.get_state(config)` 确认快照在不在。
- **节点返回了值但 State 没更新**：字段名拼错（多写/少写一个字母）时框架不会报错，只会多出一个没人读的新字段——State 用 TypedDict 时开启类型检查能提前发现。
- **无限循环烧 token**：条件边永远指回同一节点且没有计数器。所有环都要有显式计数（重试次数、步数）+ `recursion_limit` 双保险。
- **SqliteSaver 报错 `check_same_thread`**：sqlite3 连接要显式 `check_same_thread=False` 再传给 `SqliteSaver`，官方示例就是这么写的。
- **interrupt 后恢复不生效**：恢复时必须用同一个 `config`（同一个 thread_id），且传 `Command(resume=...)` 而不是重新 invoke 一份初始 State，后者会开一条新时间线。

## 7. 常见误区

- 把所有逻辑塞进一个节点：节点拆得越细，检查点、重试、观测的粒度越细；一个「大节点」等于把框架退化成普通函数调用。
- 在节点里直接改 State 对象：框架靠「返回值」收集更新，原地修改可能被 channel 合并逻辑吞掉，也让轨迹无法复盘。
- 以为 checkpointer 是业务数据库：它存的是执行状态快照，会越积越大，需要定期清理旧 thread；业务数据（章节、实体）还是进自己的表。
- 图画得越复杂越好：图只是编排层。校验、计算、权限这类确定性逻辑写普通函数被节点调用，别为了「图化」把简单流程画成迷宫。

## 8. 性能优化

- State 要瘦：只放流程推进必需的数据。大文本（整章正文、长上下文）放外部存储，State 里只留 ID 和摘要，否则每个检查点都要序列化一份全量副本，步骤一多就爆。
- 并行节点真并行：无依赖的节点（如同时查时间线和查人物关系）挂在同一超步里并行执行，比串行边省一整轮模型/IO 等待。
- 模型调用是主要成本：给每个节点单独选模型（抽取用小模型、终裁用大模型），并在节点内做结果缓存——同一输入不要因为上游重跑就再调一次模型。
- 检查点保留策略：只保留最近 N 个 thread 的快照，批处理任务跑完即归档，避免 SQLite 文件无限膨胀拖慢读写。

## 9. 版本差异

- LangGraph 已发布 1.x，API 基本稳定：`StateGraph` / `add_messages` / `ToolNode` / `tools_condition` / `interrupt` + `Command(resume=...)` 是当前推荐写法；早期教程里的 `MessageGraph`、`interrupt_before` 硬编码暂停等方式已不推荐，照旧教程抄代码容易踩空。
- 预制 Agent（如 `create_react_agent`）在新版本中逐步迁往 LangChain 的 agent 抽象下维护，自己拼图（StateGraph 手动连边）不受影响，且更利于理解原理，建议以手拼为主。
- 来源：LangGraph 官方文档与 API 参考（StateGraph、checkpoint、interrupt 章节），2026-10 核对。

## 10. 面试与自测

1. LangGraph 的 State、Node、Edge、Checkpointer 分别解决什么问题？用流程引擎的概念怎么类比？
2. 为什么消息列表字段要写 `Annotated[list, add_messages]`？不写会发生什么？
3. 图里出现环时，你靠哪三层机制保证它一定停得下来？
4. 检查点恢复时节点会被重跑，这个事实对节点实现提出了什么要求？（答：幂等）
5. 子图共享 State 和独立 State 两种接法，各适合什么场景？

### 参考答案

1. State 解决全图状态的共享与传递，类比 Flowable 的流程变量；Node 解决步骤拆分，每个节点是一个处理步骤（调模型、调工具、写库、校验），类比服务任务节点；Edge 解决节点间流转与条件分支，类比顺序流/排他网关；Checkpointer 解决状态持久化，每步后存 State 快照，支持断点续跑和回到某一步重来，类比流程实例持久化。

2. State 字段默认是覆盖语义，节点返回值会整体替换旧值。消息列表不写 Annotated[list, add_messages]，每轮返回的新消息会覆盖整个历史，之前对话全部丢失；写了之后 add_messages 作为 reducer 把新消息追加到旧列表后，历史才得以保留。这是新手最常踩的坑。

3. 三层：①环本身要有显式退出条件（步数、重试次数或明确的完成标志）；②显式计数器（如 retry_count，章节分析图里校验失败重试上限 2 次），条件边永远指回同一节点且没有计数器就会无限循环烧 token；③recursion_limit 兜底（默认 25 步，超限直接抛错）。前两层是业务层面的刹车，最后一层是框架层面的熔断。

4. 节点必须幂等：恢复时会重跑当前步，写库类节点若不做去重，恢复一次就会多写一份数据。做法是写前按业务键查重，如章节 ID + 版本，同一版本已落库就不再重复写入。

5. 共享 State：子图直接读写父图 State，简单直观，适合规模小、耦合紧的子任务；独立 State：子图定义自己的 State 再做映射，隔离更好，适合 Supervisor 模式下多个专职 Agent（抽取、矛盾检测、文风检查）并行分工，代价是调试成本更高。

## 11. 关联知识

- Agent：LangGraph 是 Agent 四种设计模式（ReAct、Plan-and-Execute、Reflection、Multi-Agent）的工程化载体。
- Tool-Calling：ToolNode 本质是把工具调用循环标准化，工具本身的定义规范见 Tool-Calling 篇。
- MCP：图里的节点可以是 MCP 客户端——通过 MCP 发现的工具同样能挂进 ToolNode，协议细节见 MCP 篇。
- 前置知识：先会裸写 ReAct 循环（Agent 篇），再学图，否则容易只会连线不懂循环为什么存在。
