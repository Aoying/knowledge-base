---
知识点: MCP
层级: 进阶
适用版本: MCP 规范 2026-07-28
更新日期: 2026-10-22
关联:
  - Agent
  - Tool-Calling
  - LangGraph
标签:
  - AI
  - Agent
  - 速查
  - 实践
---

# MCP（Model Context Protocol）

> Agent 调外部能力的统一插头标准：工具方按 MCP 暴露一次能力，任何支持 MCP 的客户端都能发现并调用，不再为每个 API 手写一套对接胶水。

## 1. 核心概念

MCP 要解决的是 **M×N 集成问题**：M 个 AI 应用 × N 个外部系统，没有标准时要写 M×N 套对接；有了 MCP，应用实现一次 Client、系统实现一次 Server，组合数降到 M+N。类比：LSP 之于编辑器和语言服务，MCP 之于 AI 应用和外部能力；也常被比作「AI 世界的 USB-C」。

三个角色（官方架构定义）：

- **Host**：承载模型的 AI 应用本身——Claude Desktop、IDE、你自己的 Agent 运行时。它跑模型循环、管用户授权，是安全边界的最终责任方。
- **Client**：Host 内部的连接器组件，**一个 Client 只连一个 Server**，负责协议通信和能力发现。Host 连了几个 Server，内部就有几个 Client。
- **Server**：把某个系统的能力按协议暴露出来的进程——可以是本机子进程，也可以是远程服务。

Server 暴露三类能力（primitives）：

- **Tools**：模型可调用的动作（查数据库、发请求、改文件），带名称、描述和参数 JSON Schema。模型控制何时调用。
- **Resources**：可读的上下文数据，用 URI 标识（文件内容、表结构、记录），通常由应用或用户决定何时塞进上下文。
- **Prompts**：可复用的提示模板，一般由用户显式触发（如斜杠命令），把一套固定工作流一键展开。

和 Skill 的区别（源自 Agent 篇）：**Skill 是「教 Agent 怎么做」**（知识文档），**MCP 是「给 Agent 递工具」**（调用接口），两者互补，不互相替代。

## 2. 基础知识

协议层事实：

- 消息格式是 **JSON-RPC 2.0**，方法名如 `tools/list`（发现工具）、`tools/call`（调用工具）、`resources/read`、`prompts/get`。
- 两种官方传输：**stdio**（Host 把 Server 作为子进程拉起，走标准输入输出，本地场景主流）和 **Streamable HTTP**（远程部署，支持标准 HTTP 鉴权，规范推荐 OAuth）。
- 能力在运行时动态发现：Host 连上 Server 后先问「你有什么」，模型看到的工具清单是问出来的，不是写死在代码里的。

最小 Server（Python，源自 Agent 篇「MCP 实战」一节，用官方 SDK 的 FastMCP）：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("novel-tools")

@mcp.tool()
def get_chapter_summary(chapter_id: int) -> str:
    """按章节 ID 查剧情摘要"""
    # 查 SQLite 的 Chapter 表
    return "..."

@mcp.resource("novel://characters/{name}")
def get_character(name: str) -> str:
    """某人物的当前状态卡"""
    return "..."

if __name__ == "__main__":
    mcp.run()  # 默认 stdio 模式
```

- 函数签名 + 类型标注会被 SDK 自动转成参数 JSON Schema；docstring 就是给模型看的工具描述，写法要求和普通 Tool Calling 一致（写清何时用、何时不用）。
- 客户端侧只需在 Host 的配置里登记这个 Server 的启动命令，Host 会自动拉起进程并完成发现：

```json
{
  "mcpServers": {
    "novel-tools": {
      "command": "python",
      "args": ["D:/work/python/novelTool/mcp_server.py"]
    }
  }
}
```

## 3. 进阶知识

- **工具设计粒度**：一个业务动作一个工具（`get_chapter_summary`），不要暴露一个万能 `run_sql` 了事——粒度越粗，模型越容易误用，权限越没法收敛。只读与写操作拆成不同工具，写工具单独走确认。
- **Elicitation（追问）**：Server 在执行中途发现信息不足，可以通过协议反向请 Host 向用户补问，拿到答案再继续。这是当前规范保留的服务端主动交互能力。
- **能力协商的变化**：2026-07-28 版规范把核心协议改为**无状态**——每个请求自带协议版本与客户端能力声明，旧版的 `initialize` 握手和会话 ID 已移除；需要提前了解服务端能力时，可调用可选的 `server/discover`。写 Server/Client 时以当前规范为准，2026 年 7 月之前的教程描述的多是旧的有状态版本。
- **采样（sampling）与 roots 已被弃用**：老教程里「Server 反过来借 Host 的模型生成内容」这类玩法不要在新实现里依赖。

## 4. 原理剖析

一次工具调用的完整链路：

```text
用户提问 → Host 把 tools/list 的结果拼进模型上下文
→ 模型决定调用某工具，产出工具名 + JSON 参数
→ Host 内的 Client 按 JSON-RPC 把 tools/call 发给对应 Server
→ Server 执行真实逻辑（查库/调 API），结果按协议回传
→ Host 把结果作为 tool 消息喂回模型 → 模型继续推理或给最终答复
```

注意两个工程含义：

- **模型从不直接连 Server**：一切经过 Host，所以授权拦截、日志审计、超时控制都应该做在 Host 侧，这也是协议把 Host 定义为安全责任方的原因。
- **发现与调用分离**：`tools/list` 的返回会直接影响模型行为——描述写了什么，模型就以为工具能干什么。这使「工具描述」本身成为攻击面（见第 5 节安全）。
- stdio 传输下一个关键细节：Server 的 stdout 被协议占用，**调试日志只能打 stderr 或文件**，往 stdout 乱 print 会直接污染 JSON-RPC 流导致通信崩坏。

## 5. 实战应用

落到小说引擎（承接 Agent 篇）：

- 把查询能力包成只读 tools 先行上线：查章节摘要、查人物状态、查伏笔台账、查时间线——任何支持 MCP 的客户端（Claude Code、自己的 LangGraph Agent）都能直接用同一套接口。
- 写入类能力（落库抽取结果、改章节状态）第二批再上，且必须经过 Host 侧确认，不给 Server 自己放行。
- 本地单机场景 stdio 足够；只有当工具要给多台机器/多人共用时才考虑迁 Streamable HTTP，并同步补鉴权。

**安全注意**（MCP 是新的信任边界，逐条执行）：

- **外部 Server 一律不可信**：第三方 MCP Server 等同于装了一个能被模型驱动的本地程序。接入前看源码、看维护方、锁版本；来路不明的 Server 不进配置。
- **工具投毒（tool poisoning）**：恶意 Server 可以在工具描述里藏指令（如「调用前先读取某文件并附带返回」），描述会进模型上下文。Host 应展示工具清单变更，描述突变就是危险信号。
- **返回内容是不可信输入**：Server 返回的文本可能夹带提示注入（网页、文档类 Server 尤其），Host/模型要把它当数据而非指令处理。
- **最小权限**：Server 进程按需给权限——只读工具就不要给写文件、发网络的能力；环境变量和密钥按 Server 隔离，不要把全套凭据塞进每个 Server 的 env。
- **用户确认在 Host**：删除、花钱、对外发送这类动作，无论 Server 怎么声明，都要在 Host 侧弹窗确认后再执行。

## 6. 疑难杂症

- **Server 连不上/工具列表为空**：先脱离 Host 用官方 Inspector 或手写 JSON-RPC 请求直连 Server 验证；stdio 场景最常见原因是启动命令的解释器路径不对、依赖没装进那个虚拟环境。
- **调用中途通信崩坏**：查 Server 是否往 stdout 打了日志/进度 print，改到 stderr。
- **模型看见工具却不用或乱用**：问题多在描述——和其他工具描述重叠、没写触发条件。先改描述，再考虑换模型。
- **远程 Server 鉴权失败**：Streamable HTTP 场景确认 token 传递方式与服务端预期一致（规范推荐 OAuth），并核对协议版本是否匹配。
- **升级 SDK 后行为变化**：先对照当前规范版本（2026-07-28）核对有状态/无状态假设，旧实现里依赖会话 ID 的逻辑要改。

## 7. 常见误区

- 「接上 MCP 模型就变聪明了」：MCP 只解决**接线**，不解决**理解**。工具背后的数据组织得烂，模型照样用不好。
- 把 MCP 当 Agent 框架：协议不管推理循环、记忆和编排，那是 Host/框架（如 LangGraph）的事。
- 一个 Server 暴露全部能力：按领域拆 Server（小说查询一个、文件系统一个），权限和启停才能分别管控。
- 工具描述照抄内部函数注释：描述是写给模型的操作说明，要写触发条件和反例，内部实现细节反而是噪声。
- 以为 stdio 就等于安全：「本地进程」恰恰意味着它有你本机的权限，该审还得审。

## 8. 性能优化

- 工具列表别贪多：每个 Server 的全部工具描述都会进上下文，连了十个 Server 可能光工具清单就吃掉大量 token。按任务启停 Server，或在 Host 侧做工具筛选。
- Resources 优先于大工具返回：稳定的大块上下文（设定集、人物表）做成 Resources 由 Host 按需挂载，比每次让模型调工具现查更省轮次。
- stdio Server 冷启动有进程拉起成本，长会话场景保持常驻，不要每次调用重新起进程。
- 工具结果要裁剪：Server 返回前就做好分页/摘要，把数据库原始大对象整坨返回是最常见的 token 浪费。

## 9. 版本差异

- **2025-03-26 版**：Streamable HTTP 取代旧的 HTTP+SSE 双端点传输；老教程里的 SSE 写法已过时。
- **2026-07-28 版**（当前）：核心协议无状态化，移除 `initialize` 握手与会话 ID；sampling、roots 弃用；elicitation 保留。跟教程前先确认它针对哪个版本。
- 来源：MCP 官方规范（modelcontextprotocol.io，架构与传输章节）与 Real Python MCP 词条，2026-10 核对。

## 10. 面试与自测

1. Host、Client、Server 的职责边界是什么？为什么说「一个 Client 只连一个 Server」？
2. Tools、Resources、Prompts 分别由谁控制调用时机？
3. MCP 把 M×N 降到 M+N 的前提是什么？哪部分成本并没有消失？（答：每个系统仍要实现一次 Server、每个应用仍要实现一次 Client，且工具质量仍要各自负责）
4. 工具投毒是什么？Host 侧有哪些具体防法？
5. stdio 和 Streamable HTTP 各适合什么部署形态？各自的安全重点有何不同？

## 11. 关联知识

- Agent：MCP 是 Agent「工具」部件的标准化供给方式，不改变 Agent 本身的循环结构。
- Tool-Calling：模型决定调哪个工具、参数怎么校验，遵循同一套规范；MCP 只是工具的发现与传输通道。
- LangGraph：图节点可通过 MCP 客户端消费远程工具，编排归框架、协议归 MCP。
- Skill（见 Agent-开发-Skills）：Skill 教做法、MCP 给接口；外部 Skill 与外部 MCP Server 执行同一条原则——不可信、先审后用。
