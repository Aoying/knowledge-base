# Agent 开发相关 Skills

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
