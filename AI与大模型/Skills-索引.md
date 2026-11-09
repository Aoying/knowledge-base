---
知识点: Skills 索引
层级: 基础
适用版本: 通用
更新日期: 2026-11-09
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

- [RAG 相关](AI与大模型/RAG-Skills.md)：rag-skills、agentic-rag、rag-eval-workflow、Shubhamsaboo/awesome-llm-apps（100+ Agent Skills 与 RAG 应用合集）
- [Agent 开发](AI与大模型/Agent-开发-Skills.md)：langgraph-agent-stack、awesome-agent-skills、awesome-agent-factories、addyosmani/agent-skills、langchain-ai/langchain-skills（LangChain/LangGraph/Deep Agents 官方技能）、muratcankoylan/Agent-Skills-for-Context-Engineering（上下文工程/多 Agent/记忆系统）、cloudflare/security-audit-skill（多阶段安全审计 skill，独立核验的机器可读审计结论）、google/skills（Google 官方 192 个 Agent Skills：Gemini API/Agent Platform/Genkit）、github/github-mcp-server（GitHub 官方 MCP Server：约 80 个工具，远程端点零本地进程）、mattpocock/skills（给真正工程师的工程 Skills：spec/TDD/评审/排障，39 个）、wshobson/agents（多工具通用插件/Agent/Skill 大库，185 个 SKILL.md）、VoltAgent/awesome-agent-skills（1000+ Agent Skills 索引）、github/awesome-copilot（GitHub 官方 Copilot 指令/Agent/Skill 库，475 个 SKILL.md）、hesreallyhim/awesome-claude-code（Claude Code 生态资源总目录）
- [开发工作流](AI与大模型/开发工作流-Skills.md)：cyberfly agent-skills（git/checkpoint/session）、obra/superpowers（TDD 开发方法论）、vercel-labs/agent-skills（React 最佳实践/设计规范）、Leonxlnx/taste-skill（前端设计品味/审美）
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

### 17. taste-skill（Leonxlnx/taste-skill）给 Agent 补前端设计品味

- **功能**：教编码 Agent 做有品味的前端：美术指导（art direction）、参考图、浏览器实测检查，替代"通用 AI 风"页面；主 skill 是 `design-taste-frontend`，另有配套的审美校准 skill。
- **适用场景**：小说引擎管理面板、知识库可视化等前端任务出图后"看起来 AI 味重"时，用它给 Agent 立审美标准；与 Anthropic 官方 `frontend-design`、vercel-labs `web-design-guidelines` 互补（官方管规范，它管品味）。
- **来源**：https://github.com/Leonxlnx/taste-skill（公开仓库，另有官网 https://tasteskill.dev）
- **热度/更新**：106k+ stars（2026-10-31 核验 GitHub API），2026-10-30 仍在更新。
- **许可证**：MIT。
- **安全注意**：只收集不安装；skill 本身是设计指令，另有浏览器检查步骤——引入前通读 SKILL.md，确认浏览器检查只读本地页面、不上传截图与源码到外部服务。

### 18. Agent-Skills-for-Context-Engineering（muratcankoylan/Agent-Skills-for-Context-Engineering）上下文工程与多 Agent 技能集

- **功能**：17 个 Agent skills 的合集，围绕上下文工程：context-fundamentals（上下文基础）、context-compression（上下文压缩）、context-optimization（上下文优化）、context-degradation（上下文退化）、memory-systems（记忆系统）、multi-agent-patterns（多智能体模式）、harness-engineering（Agent 骨架工程）、evaluation（评测）、self-managed-context（自管理上下文）、tool-design（工具设计）等；另有 researcher、template、examples 与治理文档。
- **适用场景**：ao 的小说引擎长篇连贯性痛点直接对口——多章上下文组织/压缩/裁剪、记忆系统、长程任务的上下文退化，都是这套 skills 的主题；与知识库已有的《上下文工程》篇、《Agent-记忆系统》篇互补，可作下一步深化那两篇的参考。
- **来源**：https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering（公开仓库）
- **热度/更新**：18k+ stars（2026-11-03 核验 GitHub API），2026-11-03 仍在更新。
- **许可证**：MIT。
- **安全注意**：只收集不安装；合集以指令 Markdown 为主，另有 researcher 与 template 目录——引入单个 skill 前通读其 SKILL.md，确认无敏感信息读取、网络上传、危险命令；harness 类 skill 涉及 Agent 脚手架时尤其注意权限范围。

### 19. awesome-llm-apps（Shubhamsaboo/awesome-llm-apps）100+ AI Agent、Agent Skills 与 RAG 应用合集

- **功能**：100+ 开源 AI Agent、Agent Skills 与 RAG 应用：agent_skills 目录含 9 个 skill（advisor-orchestrator-worker、commit-archaeologist、dependency-doctor、evals、first-reader、project-graveyard、scope-creep-detector、self-improving-agent-skills、thinking-out-loud），另有 100+ RAG/Agent 应用模板（Agentic RAG、知识图谱 RAG、EmbeddingGemma 本地 Agentic RAG 等）；每个 skill 带真实代码并过安全 + 评测 CI 门禁。
- **适用场景**：ao 的转型线直接对口——LangGraph Agentic RAG、混合检索 + Rerank 落地、Agent 评测（evals skill 与 RAGAS 思路对照）；本地 Agentic RAG（EmbeddingGemma）与 Ollama 路线一致；小说引擎沙盒推演可参考其多 Agent 编排模板。
- **来源**：https://github.com/Shubhamsaboo/awesome-llm-apps（公开仓库）
- **热度/更新**：143k+ stars（2026-11-04 核验 GitHub API），2026-11-04 仍在更新。
- **许可证**：Apache-2.0。
- **安全注意**：只收集不安装；合集含可执行代码与应用模板，引入单个 skill/模板前通读其代码，确认无敏感信息读取、网络上传、危险命令；`npx skills add` 只装挑中的单个 skill，不整库装。

### 20. security-audit-skill（cloudflare/security-audit-skill）多阶段安全审计 skill

- **功能**：编码 Agent 用的安全审计 skill：多阶段对抗式审计流程，审计结论经独立核验、以机器可读格式输出（报告含证据链而非笼统定性）；与子 Agent 分工做审计、发现问题与复核结论分离。
- **适用场景**：ao 的 Java 后端代码审计与 Agent 安全方向直接对口——神农 e 贷这类金融项目做代码安全自查、审计结论可机读进后续修复流程；与知识库已有的《提示注入与Agent安全》篇、《限流熔断与降级》篇互补，可作安全审计方法论的参考；已收录的第 14 项 trailofbits/skills 偏审计方法合集，本项是 Cloudflare 官方的单一聚焦 skill。
- **来源**：https://github.com/cloudflare/security-audit-skill（公开仓库）
- **热度/更新**：60k+ stars（2026-11-05 核验 GitHub API），2026-06 创建，2026-11-05 仍在更新。
- **许可证**：MIT。
- **安全注意**：只收集不安装；审计类 skill 会要求读取代码并可能调用外部工具，引入前通读其 SKILL.md，确认读取范围、网络上传与命令权限；审计结论须人工复核，不能直接当修复依据。

### 21. skills（google/skills）Google 官方 192 个 Agent Skills 合集

- **功能**：Google 官方 Agent Skills 仓库，共 192 个 SKILL.md（skills/ 下 187 个、plugins/ 下 5 个，2026-11-08 核验），分 ads、analytics、cloud、developers、identity 五类；cloud 类占大头：Gemini API、Gemini Agents API、Agent Platform（部署/推理/评测/提示管理/RAG 引擎/技能注册）、BigQuery、GKE、Genkit（Dart/Go/JS/Python）、Cloud Run、Spanner 等；developers 类含 finding-google-skills（检索本仓库 skill 的入口 skill）。
- **适用场景**：ao 的 AI Agent 转型线对口——Gemini API / Agent Platform / Genkit 的官方操作指引，可作 LangGraph 之外另一条 Agent 落地路线的参考；与知识库已有的《Agent-开发-Skills》篇互补。注意本仓库以 GCP 云服务为主，ao 的本地/Ollama 路线用不上其中多数云 skill，挑 Gemini API 与 Agent Platform 相关条目参考即可。
- **来源**：https://github.com/google/skills（公开仓库）
- **热度/更新**：23k+ stars（2026-11-08 核验 GitHub API），2026-03-31 创建，最近推送 2026-11-07。
- **许可证**：Apache-2.0。
- **安全注意**：只收集不安装；仓库以指令 Markdown 为主，引入单个 skill 前通读其 SKILL.md，确认无敏感信息读取、网络上传、危险命令；GCP 相关 skill 涉及云上资源操作，勿直接照抄命令到生产账号。

### 22. github-mcp-server（github/github-mcp-server）GitHub 官方 MCP Server

- **功能**：GitHub 官方的 Model Context Protocol 服务器，让 Agent 直接读写 GitHub：仓库、Issue、PR、Actions、代码与安全扫描等约 80 个工具、20 个工具集；提供 GitHub 托管的远程端点（https://api.githubcopilot.com/mcp/，OAuth/PAT 登录、零本地进程）与本地 Docker/Go 二进制两种用法。
- **适用场景**：ao 的知识库就在 GitHub 上、又在学 MCP：Agent 接上它就能读仓库、提 PR、查 Actions 日志、做代码扫描；与知识库已有的《MCP》篇、《Agent-开发-Skills》篇互补。注意写操作要走有作用域的 token，只读先行、写权限按需给。
- **来源**：https://github.com/github/github-mcp-server（公开仓库）
- **热度/更新**：34,319 stars（2026-11-08 核验 GitHub API），2025-03-04 创建，最近推送 2026-11-06。
- **许可证**：MIT。
- **安全注意**：只收集不安装；MCP 服务器会拿 token 操作 GitHub，引入前确认 token 作用域只给需要的仓库与权限，写/admin 权限按需授予；远程端点用 OAuth 登录，勿把 PAT 写进客户端配置文件明文。

### 23. skills（mattpocock/skills）“给真正工程师的”工程 Skills 合集

- **功能**：Matt Pocock（TypeScript 教育者）个人 .agents 目录里的工程 skills，39 个 SKILL.md（2026-11-08 核验），全是工程方法论：code-review、tdd、implement-spec、to-spec、diagnosing-bugs、grill-with-docs（Agent 先访谈你、把需求问清再动手）、codebase-design、improve-codebase-architecture、research、retro、triage、pr 等；不讲花活，只讲规划、测试、评审、排障的硬流程。
- **适用场景**：ao 的小说引擎长期迭代与 Java 后端开发纪律直接对口——“先出 spec 再实现”（implement-spec/to-spec）、TDD、code-review、排障（diagnosing-bugs）这套流程可直接套到小说引擎的开发节奏上；与已收录的第 4 项 obra/superpowers（同样是 TDD 方法论）互补：superpowers 偏流程框架，本项偏“工程师个人实战动作”，grill-with-docs 的“先访谈再动手”尤其适合需求不清时先问清再写代码。
- **来源**：https://github.com/mattpocock/skills（公开仓库）
- **热度/更新**：318,041 stars（2026-11-08 核验 GitHub API），2026-02-03 创建，最近推送 2026-11-05；截至 2026 年 11 月初是 GitHub 上星数最高的 Agent Skills 仓库。
- **许可证**：MIT。
- **安全注意**：只收集不安装；方法论类 skill 只含流程指令，风险低；引入单个 skill 前通读其 SKILL.md，确认无敏感信息读取、网络上传、危险命令。

### 24. agents（wshobson/agents）多工具通用的插件/Agent/Skill 大库

- **功能**：Claude Code、Codex CLI、Cursor、OpenCode、Antigravity CLI、GitHub Copilot、Pi 七个工具通用的插件、Agent、Skill 与命令库，一次用 Markdown 写成、再由生成器适配各工具（带校验与评测）；185 个 SKILL.md（2026-11-08 核验），按插件分领域组织，覆盖开发、测试、文档、运维等常见任务。
- **适用场景**：ao 若换 Agent 工具（Claude Code / Cursor / Copilot）不用重写 skill——同一套 Markdown 靠生成器适配多端；与已收录的第 1 项 anthropic/skills（官方 19 个生产级 skill）、第 23 项 mattpocock/skills（39 个工程方法论）互补：本项胜在“量大 + 跨工具”，挑单个插件按需取用，不要整库装。
- **来源**：https://github.com/wshobson/agents（公开仓库）
- **热度/更新**：42,295 stars（2026-11-08 核验 GitHub API），2025-07-24 创建，最近推送 2026-11-05。
- **许可证**：MIT。
- **安全注意**：只收集不安装；大库里单个 skill/plugin 的质量参差不齐，引入前通读其 SKILL.md 与插件清单，确认无敏感信息读取、网络上传、危险命令；生成器产物要核对生成结果再用。

### 25. awesome-agent-skills（VoltAgent/awesome-agent-skills）1000+ Agent Skills 索引

- **功能**：收录 1000+ 个 Agent Skill 的索引清单（README 式索引，本身不含 SKILL.md 文件）：来源包括官方开发团队与社区，覆盖 Claude Code、Codex、Gemini CLI、Cursor 等多个工具；按场景分类，可当作“找 skill 先查的目录”。
- **适用场景**：ao 要给某个具体任务找现成 skill 时，先查这份索引定位候选、再去源仓库通读 SKILL.md；与第 24 项 wshobson/agents（可直接取用的大库，185 个 SKILL.md）互补：本项是“目录”、24 项是“仓库”，一个用来发现、一个用来取用；与第 1 项 anthropic/skills（官方 19 个）对照：官方的先看，社区的按需挑。
- **来源**：https://github.com/VoltAgent/awesome-agent-skills（公开仓库）
- **热度/更新**：36,967 stars（2026-11-08 核验 GitHub API），2025-10-28 创建，最近推送 2026-11-05。
- **许可证**：MIT。
- **安全注意**：只收集不安装；索引里的 skill 来自多方、质量与权限要求各不相同，引入前通读其 SKILL.md，确认无敏感信息读取、网络上传、危险命令；不要整批拉取。

### 26. awesome-copilot（github/awesome-copilot）GitHub 官方 Copilot 指令/Agent/Skill 库

- **功能**：GitHub 官方组织下的社区共建库：Copilot 用 instructions（232 个）、agents（249 个）、skills（475 个 SKILL.md，2026-11-08 核验），另有 cookbook、extensions 与 hooks；覆盖开发、测试、文档、评审等场景的现成提示词与工作流。
- **适用场景**：ao 在 VS Code / GitHub Copilot 里干活时直接取用——instructions 放 `.github/instructions/`、agents 与 skills 按需取单个文件；与第 1 项 anthropic/skills（Claude Code 官方）、第 24 项 wshobson/agents（多工具大库）、第 25 项 VoltAgent 索引（发现目录）互补：本项是 Copilot 一侧的官方入口，补上了索引里 Copilot 专项的缺口。
- **来源**：https://github.com/github/awesome-copilot（公开仓库）
- **热度/更新**：40,913 stars（2026-11-08 核验 GitHub API），2025-06-11 创建，最近推送 2026-11-06。
- **许可证**：MIT。
- **安全注意**：只收集不安装；社区共建内容质量参差不齐，引入前通读单个 instruction/skill 文件，确认无敏感信息读取、网络上传、危险命令；不要整库装，按任务取单个文件。

### 27. awesome-claude-code（hesreallyhim/awesome-claude-code）Claude Code 生态资源总目录

- **功能**：Claude Code 生态的精选资源总目录（README 式索引，本身不含 SKILL.md）：skills、hooks、slash commands、agents、编排器、插件与开发工具，按类别整理并附 CSV 清单；是“找 Claude Code 相关 skill/插件先查的目录”。
- **适用场景**：ao 要给某个具体任务找 Claude Code 的 skill 或插件时，先查这份目录定位候选、再去源仓库通读 SKILL.md；与第 25 项 VoltAgent/awesome-agent-skills（1000+ 跨工具索引）互补：本项只盯 Claude Code 一侧、更深；与第 1 项 anthropic/skills（官方 19 个）对照：官方的先看，社区的按需挑。
- **来源**：https://github.com/hesreallyhim/awesome-claude-code（公开仓库）
- **热度/更新**：57,516 stars（2026-11-08 核验 GitHub API），2025-04-19 创建，最近推送 2026-11-08。
- **许可证**：未声明（资源清单类仓库，本身不含可执行代码）。
- **安全注意**：只收集不安装；目录里的 skill/plugin 来自多方、质量与权限要求各不相同，引入前通读其 SKILL.md，确认无敏感信息读取、网络上传、危险命令；不要整批拉取。

### 28. agent-skills（addyosmani/agent-skills）生产级工程 Skills 合集

- **功能**：Addy Osmani（Google Chrome 团队）维护的 25 个生产级工程 skills：API 与接口设计、TDD、规范驱动开发、代码评审与质量、调试与错误恢复、可观测性与插桩、安全加固、性能优化、Git 工作流、CI/CD、文档与 ADR、上下文工程、增量实现等；每个 skill 一个 SKILL.md，面向 AI 编程 Agent 直接取用。
- **适用场景**：ao 的小说引擎长期迭代与 Java 后端开发纪律：TDD、评审、排障、性能优化这几条可直接当工作流范本；与第 4 项 obra/superpowers（方法论）、第 23 项 mattpocock/skills（工程 Skills，39 个）互补：本项更偏“单个任务怎么做”的实操 skill，三个合起来覆盖方法论与实操。
- **来源**：https://github.com/addyosmani/agent-skills（公开仓库）
- **热度/更新**：114,312 stars（2026-11-09 核验 GitHub API），2026-02-15 创建，最近推送 2026-11-01。
- **许可证**：MIT。
- **安全注意**：只收集不安装；引入前通读要用的单个 SKILL.md，确认无敏感信息读取、网络上传、危险命令；不要整库装，按任务取单个文件。
