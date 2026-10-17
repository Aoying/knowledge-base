# Skills 索引

> 每晚 20:30 自动搜集更新。只收集分析，不安装不执行。
> 格式约定：每个 skill 记录名称、功能、适用场景、来源、许可证、更新时间、安全注意事项。

## 分类索引

- [RAG 相关](01-AI与Agent/RAG-Skills.md)：rag-skills、agentic-rag、rag-eval-workflow
- [Agent 开发](01-AI与Agent/Agent-开发-Skills.md)：langgraph-agent-stack、awesome-agent-skills、awesome-agent-factories、addyosmani/agent-skills
- [开发工作流](01-AI与Agent/开发工作流-Skills.md)：cyberfly agent-skills（git/checkpoint/session）、obra/superpowers（TDD 开发方法论）、vercel-labs/agent-skills（React 最佳实践/设计规范）

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
- **安全注意**：方法论类 skill 只含流程指令，风险低；落地前通读 SKILL.md 确认无隐藏网络上传或敏感文件读取。详见 [开发工作流 Skills](01-AI与Agent/开发工作流-Skills.md)。

### 5. agent-skills（vercel-labs/agent-skills）

- **功能**：Vercel 官方前端工程 skills：`react-best-practices`、`web-design-guidelines`。
- **适用场景**：小说引擎管理面板 / 知识库可视化等前端任务，配合 Anthropic `frontend-design`（管审美）一起用。
- **来源**：https://github.com/vercel-labs/agent-skills
- **热度/更新**：32k+ stars（2026-10-09），官方背书，维护活跃。
- **安全注意**：只收集不安装；引入前检查 SKILL.md 是否有外部网络请求。详见 [开发工作流 Skills](01-AI与Agent/开发工作流-Skills.md)。

## 待评估

- （夜间任务持续补充）
