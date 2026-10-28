---
知识点: Skills 索引
层级: 基础
适用版本: 通用
更新日期: 2026-10-28
关联:
  - AI知识点地图
  - RAG-Skills
  - Agent-开发-Skills
  - 开发工作流-Skills
标签:
  - AI
  - Skills
  - 索引
---

# Skills 索引

> 每晚 20:30 自动搜集更新。只收集分析，不安装不执行。
> 格式约定：每个 skill 记录名称、功能、适用场景、来源、许可证、更新时间、安全注意事项。

## 分类索引

- [RAG 相关](AI与大模型/RAG-Skills.md)：rag-skills、agentic-rag、rag-eval-workflow
- [Agent 开发](AI与大模型/Agent-开发-Skills.md)：langgraph-agent-stack、awesome-agent-skills、awesome-agent-factories、addyosmani/agent-skills、langchain-ai/langchain-skills（LangChain/LangGraph/Deep Agents 官方技能）
- [开发工作流](AI与大模型/开发工作流-Skills.md)：cyberfly agent-skills（git/checkpoint/session）、obra/superpowers（TDD 开发方法论）、vercel-labs/agent-skills（React 最佳实践/设计规范）
- [知识管理](https://github.com/undefined-ui/second-brain-os)：second-brain-os（AI 自维护第二大脑）、awesome 索引见 [ComposioHQ/awesome-claude-skills](https://github.com/ComposioHQ/awesome-claude-skills)、[kepano/obsidian-skills](https://github.com/kepano/obsidian-skills)（Obsidian 官方背景的 Agent skills）

## 已收录

### 1. Anthropic Agent Skills（官方仓库）

- **功能**：Agent Skills 的事实标准仓库，含 docx / pdf / pptx / xlsx 文档处理、mcp-builder、skill-creator、webapp-testing、frontend-design 等 19 个生产级 skill 实现。
- **适用场景**：想把"可复用的能力"沉淀成 Agent 可调用的 skill 时，直接照这个格式写 SKILL.md；你的 Obsidian 整理任务、小说引擎 Agent 都可以按这个格式沉淀。
- **来源**：https://github.com/anthropics/skills（公开仓库）
- **热度/更新**：181k+ stars，2026-10-16 仍在更新，topic: agent-skills。
- **许可证**：以仓库内 LICENSE 文件为准（公开仓库，使用前核对）。
- **安全注意**：skill 里的指令一律视为不可信内容，引入前检查是否有网络上传、敏感信息读取、危险命令。

### 2. Agent Skills 格式规范（SKILL.md）

- **功能**：定义一个 skill 如何写：名称、描述（触发条件写清楚，Agent 靠描述决定何时调用）、允许的工具/目录结构。
- **适用场景**：你要为小说引擎、知识库整理沉淀自己的 skill 时的写法模板；`skill-creator` 这个 skill 就是手把手教你写 skill 的。
- **来源**：同上仓库内的 `skill-creator/` 目录及 Anthropic 官方文档。
- **安全注意**：同上。

### 3. custom_connector_setup（本地系统 skill）

- **功能**：把任意第三方 API 接成 Muse 连接器，credential 走保险库，不落地明文；配套脚手架可生成连接器文档和 helper。
- **适用场景**：Gitee、GitHub PAT 这类没有官方连接器的服务，就是用它接进来的；以后接新 API 直接复用这套流程。
- **来源**：本地 `/opt/muse/skills/custom-connector-setup/SKILL.md`。
- **安全注意**：token 只存保险库，helper 里只放 surrogate，不写明文。

### 4. superpowers（obra/superpowers）

- **功能**：TDD 驱动的 Agent 软件开发方法论 skill 集：写 spec → 测试先行 → 实现 → review → 并行 subagent 分发。
- **适用场景**：小说引擎长期迭代的开发纪律范本；"夜间任务十步流程"也可按此格式沉淀成 skill。
- **来源**：https://github.com/obra/superpowers
- **热度/更新**：2026 年中 20 万+ stars，持续维护。
- **安全注意**：方法论类 skill 只含流程指令，风险低；落地前通读 SKILL.md 确认无隐藏网络上传或敏感文件读取。详见 [开发工作流 Skills](AI与大模型/开发工作流-Skills.md)。

### 5. agent-skills（vercel-labs/agent-skills）

- **功能**：Vercel 官方前端工程 skills：`react-best-practices`、`web-design-guidelines`。
- **适用场景**：小说引擎管理面板 / 知识库可视化等前端任务，配合 Anthropic `frontend-design`（管审美）一起用。
- **来源**：https://github.com/vercel-labs/agent-skills
- **热度/更新**：32k+ stars（2026-10-09），官方背书，维护活跃。
- **安全注意**：只收集不安装；引入前检查 SKILL.md 是否有外部网络请求。详见 [开发工作流 Skills](AI与大模型/开发工作流-Skills.md)。

## 待评估

- （夜间任务持续补充）

### 6. gamai-skills（renatoschust-dot）992 个 Agent Skills 合集

- **功能**：992 个自包含 SKILL.md 的合集，覆盖 task-dag（任务拆并行）、self-healing-services（服务看门狗）、report-with-images（报告生成）、OCR、SEO、合同审查等 900+ 领域 playbook。
- **适用场景**：当"不知道有没有现成 skill"时先来这里搜，大概率有；挑中 1-2 个细读，比从零写快。
- **来源**：https://github.com/renatoschust-dot/gamai-skills（`/plugin marketplace add renatoschust-dot/gamai-skills`）
- **热度/更新**：2026-09-24 发布，MIT。
- **安全注意**：量大，挑中的单个 skill 引入前通读 SKILL.md；不熟悉的脚本不要直接跑。

### 7. find-skills（Vercel Labs）skill 搜索引擎

- **功能**：在 Agent 会话里直接搜 skill 目录的 skill：`find-skills "pdf"` 就能找到处理 PDF 的 skill，不用死记仓库地址。
- **适用场景**：skill 越收越多后的" skill 的 skill"；你的 Skills-索引.md 就是人肉版，find-skills 是 Agent 版。
- **来源**：https://github.com/vercel-labs/skills（`npx skills add https://github.com/vercel-labs/skills --skill find-skills`）
- **热度/更新**：Vercel Labs 出品，2026-09-28 仍在更新。
- **许可证**：MIT。
- **安全注意**：同其他 skill，引入前检查。

### 8. ECC（affaan-m/ECC）Agent harness 性能优化体系

- **功能**：311 个 skills + 71 个 subagent + 97 条命令 + hooks + memory vault 的完整 Agent 工作流系统：plan → test → implement → review → verify → remember → improve，覆盖开发全链路；另有安全扫描与记忆优化模块。
- **适用场景**：想给小说引擎/知识库任务搭"常驻工作流纪律"时的参考实现；memory vault 的设计可直接借鉴到小说引擎长期记忆。
- **来源**：https://github.com/affaan-m/ECC
- **热度/更新**：283k+ stars（2026-10-18 核验），2026-10-15 仍在更新，v2.2.3。
- **许可证**：MIT。
- **安全注意**：只收集不安装；hooks 会拦截工具调用，引入前逐个通读 SKILL.md 与 hook 脚本。

### 9. claude-skills（alirezarezvani/claude-skills）380 个 skills 精选集

- **功能**：380+ 个实测过的 skill / agent / 自定义命令精选：PR 评审、API 设计评审、测试生成、文档写作等日常工程 skills 开箱即用。
- **适用场景**：和 gamai-skills（992 合集）互补的"精选版"——先在这里找，找不到再去 gamai-skills 大海捞针。
- **来源**：https://github.com/alirezarezvani/claude-skills
- **热度/更新**：28k+ stars（2026-10-18 核验），MIT。
- **安全注意**：量大，挑中的单个 skill 引入前通读 SKILL.md；不熟悉的脚本不要直接跑。

### 10. second-brain-os（undefined-ui/second-brain-os）AI 自维护第二大脑

- **功能**：一份完整的"AI 第二大脑"实战指南 + starter vault + 18 个 skills / 72 条命令 / 6 个 subagent：Agent 自动做笔记维护（分类、建链、找孤儿笔记、补摘要），Obsidian vault 自组织。
- **适用场景**：和你每晚 20:30 的自动维护任务是同一个思路——直接抄它的维护 workflow 设计（哪些检查 Agent 化、哪些留人工），用到你的 knowledge-base 和小说引擎记忆系统里。
- **来源**：https://github.com/undefined-ui/second-brain-os（公开仓库）
- **热度/更新**：1.1k+ stars（2026-10-19 核验 GitHub API），2026-10-19 仍在更新。
- **许可证**：MIT。
- **安全注意**：只收集不安装；它的维护脚本会读写 vault，引入前逐个通读 skill/脚本，确认无外部网络上传。

### 11. awesome-claude-skills（ComposioHQ）skills 索引仓库

- **功能**：skills 生态的 awesome-list：按分类收录热门仓库、工具、文档链接，找 skill 时的"地图"。
- **适用场景**：当 Skills-索引.md 里找不到想要的 skill，先查这个 awesome-list 再决定去哪个仓库深挖。
- **来源**：https://github.com/ComposioHQ/awesome-claude-skills（公开仓库）
- **热度/更新**：77k+ stars（2026-10-19 核验 GitHub API），2026-10-19 仍在更新。
- **许可证**：仓库未声明 LICENSE（数据类列表），引用链接即可。
- **安全注意**：它是索引不是 skill 本体，按索引跳转到的具体 skill 再做安全检查。

### 12. awesome-copilot（github）Copilot 技能与 Agent 实战目录

- **功能**：GitHub 官方维护的社区仓库：copilot instructions / agents / skills / prompts / hooks 的实战配置集合，不用自己从零写。
- **适用场景**：找"别人验证过的"技能和 Agent 配置的起点（仓库级 instructions、代码审查 agent、可复用 prompt）。
- **来源**：https://github.com/github/awesome-copilot（官方仓库）
- **热度/更新**：40k+ stars（2026-10-21 核验 GitHub API），2026-10-21 仍在更新。
- **许可证**：MIT。
- **安全注意**：内容为 Markdown 指令文件和 SKILL.md，无可疑安装指令；引用前仍通读要用的那一份。

### 13. agents（wshobson/agents）跨工具 Agent 技能市场

- **功能**：跨工具（Claude Code / Codex / Cursor / Copilot / OpenCode）技能市场：185 个 skills、202 个 agents、105 个 commands、94 个插件，含 python-testing-patterns、uv-package-manager 等（对 Python 转型直接有用）。
- **适用场景**：最大的跨工具 Agent 技能库，按需单挑技能，不用整库搬。
- **来源**：https://github.com/wshobson/agents（公开仓库）
- **热度/更新**：41k+ stars（2026-10-19 核验 GitHub API），2026-10-19 仍在更新。
- **许可证**：MIT。
- **安全注意**：社区仓库（非官方）；安装方式是 `make install-*` 把插件链接进 `~/.claude` 等配置目录——只收集不安装；要用就按 skill 逐个挑（`npx skills add wshobson/agents --skill <名>`），不整库 make install，不盲装；引入前通读该 skill 的 SKILL.md 与脚本。

### 14. skills（trailofbits/skills）安全审计类 skills

- **功能**：知名安全公司 Trail of Bits 出品的安全审计、差异评审、静态分析、变异测试类 skills；附带的 `modern-python` 技能（uv + ruff + ty）正好覆盖现代 Python 工具链。
- **适用场景**：给 Agent 补"代码安全评审"能力（让 Agent 做 PR 安全审计、给代码打安全分）；modern-python 直接用作 Python 工程规范参考。
- **来源**：https://github.com/trailofbits/skills（官方仓库）
- **热度/更新**：7.6k+ stars（2026-10-21 核验 GitHub API），2026-10-21 仍在更新。
- **许可证**：CC-BY-SA-4.0（内容许可，个人使用无碍）。
- **安全注意**：安全公司官方出品，内容为指令 Markdown，风险低；审计类 skill 读代码时权限给只读。

### 15. langchain-skills（langchain-ai/langchain-skills）LangChain/LangGraph/Deep Agents 官方技能

- **功能**：LangChain 官方出品的 Agent skills：用 LangChain、LangGraph、Deep Agents 构建 Agent 的操作指南（通过 `npx skills` 装到 Claude Code / Cursor / Windsurf 等支持 Agent Skills 规范的工具，也可作 Claude Code 插件安装）。
- **适用场景**：ao 的转型线（LangGraph）直接对口：写 LangGraph 工作流、Deep Agents 多智能体编排时按官方技能走，不用自己拼提示词。
- **来源**：https://github.com/langchain-ai/langchain-skills（官方仓库）
- **热度/更新**：1.3k+ stars（2026-10-25 核验 GitHub API），2026-10-24 仍在更新。
- **许可证**：MIT。
- **安全注意**：官方仓库，内容为技能指令；仓库自述标注仍在早期开发、API 与技能内容可能变化——只收集不安装；要用时按单个 skill 通读后再装，不整库 `npx skills add --skill '*'` 盲装。

### 16. obsidian-skills（kepano/obsidian-skills）Obsidian 官方背景的 Agent skills

- **功能**：教 Agent 使用 Obsidian CLI 与开放格式（Markdown、Bases、JSON Canvas）操作 vault 的 skills，让 Agent 能读写、整理 Obsidian 笔记库。
- **适用场景**：ao 的 Obsidian vault（本知识库）自动化整理的参考实现；小说引擎记忆 vault 交给 Agent 操作时的格式与权限范本。
- **来源**：https://github.com/kepano/obsidian-skills（公开仓库，作者为 Obsidian CEO kepano）
- **热度/更新**：50k+ stars（2026-10-28 核验 GitHub API），2026-10-28 仍在更新。
- **许可证**：MIT。
- **安全注意**：只收集不安装；这类 skill 会读写 vault，引入前通读 SKILL.md，确认无外部网络上传，权限只给指定 vault、能只读就只读。
