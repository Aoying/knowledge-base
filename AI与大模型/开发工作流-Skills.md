---
知识点: 开发工作流 Skills
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - AI知识点地图
  - Skills-索引
标签:
  - AI
  - Skills
  - 实践
---

# 开发工作流 Skills

> Git、会话管理、代码检查、开发方法论类实用 skill。只收集分析，不安装不执行。

## 1. agent-skills（cyberfly）个人 Agent 技能集

- **功能**：面向日常开发的实用 skill：`git-staged` / `staged`（只提交本会话改过的文件，不碰用户手动改动）、`commit`（Conventional Commits 自动提交）、`stash-staged`、`checkpoint` / `load-checkpoint`（将会话上下文存成带时间戳的 handoff 文件，跨会话恢复）、`session-log`（每分支记录决策）、`pr-description`（从分支变更生成 PR 描述）、`branch-bug-check`（读 diff、跑测试，给 ranked bug 报告）。
- **适用场景**：直接套用：`staged` / `commit` 语义正是夜间任务"只推任务改动、不碰用户修改"的安全规则；`checkpoint` 是长会话防上下文丢失的现成方案。
- **来源**：https://github.com/cyberfly/agent-skills
- **热度/更新**：3 stars，2026-07-31 更新。
- **许可证**：未声明，引用思路不抄代码。
- **安装**：clone 后 `node sync-skills.js` 把 skill symlink 到 `~/.claude/skills` 等目录；`node toggle-skills.js off/on` 临时开关。
- **依赖**：Node.js。
- **安全注意**：sync 脚本会在家目录建 symlink，跑之前先读脚本确认只动 skill 目录；git 相关 skill 会执行提交类操作，只在确认的仓库用。

## 2. superpowers（obra/superpowers）TDD 驱动的 Agent 开发方法论

- **功能**：一套完全由可组合 Skills 实现的软件开发方法论：写 spec → 测试先行（TDD）→ 实现 → code review → 并行 subagent 分发。把"每个会话都要重申一遍的纪律"（先测试、要 review、secrets 不进提交）沉淀成常驻 skill，Agent 自动遵守。
- **适用场景**：小说引擎这类长期迭代项目的开发纪律；给 Agent 立规矩的范本——你的"夜间任务十步流程"也可以按这个格式写成 skill。
- **来源**：https://github.com/obra/superpowers
- **热度/更新**：2026 年中 20 万+ stars，生态安装量最大的一批 skill 集合之一，持续维护。
- **许可证**：以仓库内 LICENSE 为准（使用前核对）。
- **安装**：`npx -y skills add obra/superpowers --agent claude-code`（装到项目 `.claude/skills/`）。
- **依赖**：Claude Code / Cursor / Codex 等支持 Agent Skills 的客户端。
- **安全注意**：方法论类 skill 本身只含流程指令，风险低；但它会驱动 Agent 执行测试/提交命令，skill 落地前通读 SKILL.md 确认无隐藏的网络上传或敏感文件读取。

## 3. agent-skills（vercel-labs/agent-skills）Vercel 官方前端工程 skills

- **功能**：Vercel 官方维护的 skill 合集，含 `react-best-practices`（React/Next.js 最佳实践）、`web-design-guidelines`（Web 设计规范审计）等。帮 Agent 写出符合生产标准的 React 代码、审查现有 UI 是否合规。
- **适用场景**：小说引擎管理面板 / 知识库可视化页面这类前端任务——配合 `frontend-design`（Anthropic 官方审美 skill）一起用，一个管工程规范、一个管设计品味。见 [工程实践/前端/前端开发.md](工程实践/前端/前端开发.md)。
- **来源**：https://github.com/vercel-labs/agent-skills
- **热度/更新**：32k+ stars（2026-10-09 统计），Vercel 官方背书，维护活跃。
- **许可证**：以仓库内 LICENSE 为准（使用前核对）。
- **安装**：`npx skills add https://github.com/vercel-labs/agent-skills --skill "react-best-practices"`（按需单装）。
- **依赖**：支持 Agent Skills 的客户端；React 项目。
- **安全注意**：只收集不安装；引入前检查 SKILL.md 是否有外部网络请求或读取仓库外文件。

## 4. grill-me（mattpocock/skills）开工前拷问计划

- **功能**：在你动手写代码前，Agent 化身"魔鬼代言人"拷问你的计划：边界条件、失败模式、有没有更简单的方案。过不了拷问就不开工。
- **适用场景**：小说引擎这种长周期项目，每次大功能（如 RAG 接入、沙盒推演）开工前先被拷问一遍，比写完返工便宜得多。
- **来源**：https://github.com/mattpocock/skills（`npx skills add https://github.com/mattpocock/skills --skill grill-me`）
- **热度/更新**：Matt Pocock（知名 TS/AI 教育者）维护，2026-09-30 仍在更新。
- **许可证**：MIT。
- **安全注意**：只收集不安装；引入前检查 SKILL.md。

## 5. ECC（Everything Claude Code）Agent 全家桶

- **功能**：Claude Code 的完整 harness：68 个 agents、292 个 skills、AgentShield 安全扫描（102 条静态规则）、hooks/rules/memory、跨 Claude Code/Codex/Cursor。
- **适用场景**：superpowers 给的是"方法论"，ECC 给的是"全套装备"；团队级 Agent 落地、需要安全扫描和质量门禁时参考它的结构。
- **来源**：https://github.com/affaan-m/everything-claude-code（按 2026-09 第三方榜单约 267k stars）
- **许可证**：MIT（使用前核对仓库 LICENSE）。
- **安全注意**：量大，只借鉴结构；真要装，先读 AgentShield 规则再说。
