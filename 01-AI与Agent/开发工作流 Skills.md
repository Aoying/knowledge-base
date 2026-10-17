# 开发工作流 Skills

> Git、会话管理、代码检查类实用 skill。只收集分析，不安装不执行。

## 1. agent-skills（cyberfly）个人 Agent 技能集

- **功能**：面向日常开发的实用 skill：`git-staged` / `staged`（只提交本会话改过的文件，不碰用户手动改动）、`commit`（Conventional Commits 自动提交）、`stash-staged`、`checkpoint` / `load-checkpoint`（将会话上下文存成带时间戳的 handoff 文件，跨会话恢复）、`session-log`（每分支记录决策）、`pr-description`（从分支变更生成 PR 描述）、`branch-bug-check`（读 diff、跑测试，给 ranked bug 报告）。
- **适用场景**：直接套用：`staged` / `commit` 语义正是夜间任务"只推任务改动、不碰用户修改"的安全规则；`checkpoint` 是长会话防上下文丢失的现成方案。
- **来源**：https://github.com/cyberfly/agent-skills
- **热度/更新**：3 stars，2026-07-31 更新。
- **许可证**：未声明，引用思路不抄代码。
- **安装**：clone 后 `node sync-skills.js` 把 skill symlink 到 `~/.claude/skills` 等目录；`node toggle-skills.js off/on` 临时开关。
- **依赖**：Node.js。
- **安全注意**：sync 脚本会在家目录建 symlink，跑之前先读脚本确认只动 skill 目录；git 相关 skill 会执行提交类操作，只在确认的仓库用。
