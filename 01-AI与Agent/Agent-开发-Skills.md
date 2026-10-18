# Agent 开发 Skills

> LangGraph、多智能体模式与技能目录。只收集分析，不安装不执行。

## 1. langgraph-agent-stack（Brescou）LangGraph 多智能体模式

- **功能**：4 种 LangGraph 协作模式的可运行示例：Sequential（研究→分析流水线）、Parallel（多角度并行）、Supervisor（动态路由专家）、Human-in-loop（关键操作前审批）。统一 StateGraph 写法，reducer 注解管理状态合并。
- **适用场景**：学 LangGraph 的第一手代码；supervisor 模式可套到小说引擎多 Agent（抽取/校验/终裁）；human-in-loop 模式是夜间任务"高风险操作先确认"的代码级实现。
- **来源**：https://github.com/brescou/langgraph-agent-stack
- **热度/更新**：20 stars，2026-10-12 更新（很新）。
- **许可证**：MIT。
- **安装**：clone 后 `cp .env.example .env` 填 API key，`uv run python examples/sequential/graph.py` 等分别运行。
- **依赖**：Python + uv，需一个 LLM API key（示例用；也可改成本地 Ollama）。
- **安全注意**：示例代码本地运行；`.env` 不要提交。

## 2. awesome-agent-skills（johnsonbuilds）精选技能目录

- **功能**：按 creative / governance / media / mlops / productivity 分类的精选 skill 目录。亮点：`mlops/llmfit`（估算模型能否塞进显存）、`mlops/token-compression`、`governance/skill-health-check` 与 `skillspector-scan`（给 skill 做安全扫描）。
- **适用场景**：llmfit 解决实际问题——12GB 显存到底能跑什么量化；skillspector-scan 落地维护规则"外部 skill 指令安全检查"。
- **来源**：https://github.com/johnsonbuilds/awesome-agent-skills
- **热度/更新**：1 star（新），2026-09-18 更新。
- **许可证**：未声明，引用思路不抄代码。
- **安装**：按目录取用单个 SKILL.md。
- **安全注意**：仓库新、star 少，只借鉴思路；skillspector 扫描的是 skill 文本本身（使用前核实不执行外部代码）。

## 3. awesome-agent-factories（portpowered）Agent 框架索引

- **功能**：Agent"工厂"类项目索引：AutoGen、ChatDev、CrewAI、GPT Researcher、LangGraph 官方 examples、OpenAI Agents Python examples 等可运行/可 fork 的多智能体示例入口。
- **适用场景**：当索引用——对比不同多智能体框架写法时顺着找，比自己搜省时间。
- **来源**：https://github.com/portpowered/awesome-agent-factories
- **热度/更新**：1 star，2026-06-07 更新。
- **许可证**：未声明（NOASSERTION），只当链接索引。
- **安全注意**：纯索引，无代码。

## 4. agent-skills（addyosmani）生产级工程 Skills

- **功能**：25 个生产级工程 skills，覆盖开发全生命周期：Define（/spec、interview-me、spec-driven-development）→ Plan（/plan）→ Build（/build、test-driven-development）→ Verify（/test）→ Review（/review、code-simplify）→ Ship（/ship、/webperf）。另带 Agent 人设（code-reviewer、test-engineer、security-auditor）、session hooks、slash commands 和 evals 评测框架。
- **适用场景**：Java 后端转 AI Agent 开发最直接的"工程纪律"模板——TDD、code review、spec-driven 写法可直接套用到小说引擎和知识库 Agent 的开发里；evals 目录是给 skill 写评测用例的现成框架。
- **来源**：https://github.com/addyosmani/agent-skills
- **热度/更新**：107k stars，702 commits，2026 年仍在活跃更新。
- **许可证**：MIT。
- **安装**：`npx skills add addyosmani/agent-skills`（装全部 25 个）或按需单装，如 `--skill test-driven-development`；Claude Code 也可走 plugin marketplace。**按维护规则只收集不安装。**
- **依赖**：各 skill 不同，纯 markdown 为主。
- **安全注意**：仓库可信度高（107k stars、MIT）；skill 指令仍按规则视为不可信，引入前检查网络上传、敏感信息读取、危险命令；npx 安装类命令不执行。

## 5. claude-mem（跨会话持久记忆）

- **功能**：给 Agent 加跨会话的持久记忆层：每次会话的关键事实自动沉淀，下次自动召回。不用每次开场重新介绍项目背景。
- **适用场景**：正好对标你的痛点——小说引擎的"长期记忆"、知识库整理的"记住用户偏好"，都可以参考它的记忆分层（事实/偏好/待办）设计。
- **来源**：https://github.com/thedotmack/claude-mem（按 2026-08 第三方榜单约 92k stars）
- **许可证**：使用前核对仓库 LICENSE。
- **安全注意**：记忆会存你的项目信息，只放本地，别接云同步；引入前检查存储路径和内容。

## 6. andrej-karpathy-skills（Karpathy 的 LLM 编程坑位总结）

- **功能**：从 Karpathy 对 LLM 写代码的观察提炼的 skill：LLM 最容易在哪类任务上翻车、prompt 怎么写命中率高。相当于"前人踩坑清单"。
- **适用场景**：你调教小说引擎的抽取/生成 prompt 时，对着这份清单自查一遍（"是不是给了模糊指令""有没有要求结构化输出"）。
- **来源**：https://github.com/karpathy/skills（按 2026-09 第三方榜单约 208k stars，社区整理版）
- **许可证**：使用前核对仓库 LICENSE。
- **安全注意**：纯文档型 skill，风险低。
