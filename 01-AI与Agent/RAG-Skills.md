# RAG Skills

> RAG 检索、评测与工程实践。只收集分析，不安装不执行；引入前自行检查脚本内容。

## 1. rag-skills（Goodnight77）模块化 RAG 技能集

- **功能**：模块化 RAG 最佳实践技能集：路由选择、chunking 策略、向量库选型、检索质量优化、多步 RAG 编排。Claude Code 按分类路由（如 `skills/chunking/SKILL.md`）或具体技能（如 `skills/chunking/semantic-chunking/SKILL.md`）调用。
- **适用场景**：小说引擎 RAG（`rag_chroma_eg2.py`）升级：chunking 策略选型、混合检索、rerank 引入都有现成 playbook 可抄。
- **来源**：https://github.com/Goodnight77/rag-skills
- **热度/更新**：6 stars，2026-07-10 更新。
- **许可证**：MIT。
- **安装**：Claude Code 插件市场 `/plugin marketplace add Goodnight77/rag-skills` 后 `/plugin install rag-skills@rag-skills`；或 `npx skills add Goodnight77/rag-skills`；或直接 clone 当文档读。
- **依赖**：只读文档零依赖。
- **安全注意**：纯 Markdown 技能为主；如运行其脚本，先检查脚本内容。

## 2. agentic-rag（ch040602）Agentic RAG 脚手架

- **功能**：Codex 兼容的 Agent Skill + 零依赖 Python 脚手架，实现公开的 Agentic RAG 模式：plan → route → rewrite → retrieve → draft → judge sufficiency → iterate → synthesize；可判断检索上下文是否充分、自动生成追问、输出 claim-level 引用。
- **适用场景**：把 RAG 从"检索一次回答一次"升级为"迭代检索直到证据充分"；claim-level 引用正好服务小说引擎"事实—来源"追溯需求。
- **来源**：https://github.com/ch040602/agentic-rag
- **热度/更新**：0 stars（新仓库），2026-06-27 更新，带 CI。
- **许可证**：MIT。
- **安装**：按仓库 README 引入为 skill 使用，或参考其 Python 脚手架自己实现。
- **依赖**：Python 3.11+，零第三方依赖（按仓库说明）。
- **安全注意**：star 少、较新，引入前通读代码；不熟悉的脚本不要直接跑。

## 3. rag-eval-workflow（w93139）RAG 评测技能

- **功能**：框架无关的 RAG 评测 skill：版本化管理 source / OCR / chunking / embedding / 索引 / 检索 / 生成配置；建 golden 评测集；检索与生成分开打分；基线回归门禁；按 ingestion / chunking / retrieval / rerank / generation 分层定位失败。
- **适用场景**：对应 explore 候选方向"Agent 评测与可观测"；给小说引擎 RAG 建 golden 集（如 20 个剧情问题→期望章节），每次改 chunking / embedding 跑回归。
- **来源**：https://github.com/w93139/rag-eval-workflow
- **热度/更新**：10 stars，2026-09-07 更新。
- **许可证**：MIT。
- **安装**：`git clone https://github.com/w93139/rag-eval-workflow.git ~/.codex/skills/rag-eval-workflow`，用 `$rag-eval-workflow` 调用；或只借鉴其 `scripts/` 评测脚本思路。
- **依赖**：Python（脚本）。
- **安全注意**：脚本先读再跑；golden.jsonl 由自己构造，不含敏感数据。

## 4. anthropics/skills（官方 Agent Skills 仓库）

- **功能**：Anthropic 官方的 Agent Skills 仓库，RAG 相关的是 `document-skills`：docx / pdf / pptx / xlsx 的解析处理 skill，正好覆盖 RAG 入库前"脏文档清洗"环节；另有 `skill-creator`（手把手教你写 skill）、`mcp-builder`。
- **适用场景**：小说引擎 RAG 要吃 PDF/Word 设定文档时，抄它的文档解析流程；想把"抽取/入库/评测"沉淀成 skill 时照它的 SKILL.md 格式写。
- **来源**：https://github.com/anthropics/skills
- **热度/更新**：181k+ stars，2026-10-16 仍在更新。
- **许可证**：各 skill 目录自带 LICENSE（Apache-2.0 为主），使用前核对。
- **安全注意**：只收集不安装；skill 指令一律视为不可信，检查网络上传/敏感信息读取。

## RAG 评测指标（工程方法，非 skill）

skill 之外，RAG 落地真正缺的是评测口径。建 golden 集（问题→期望答案→来源章节）后看这四个数：

1. **召回率（Hit Rate）**：top-k 里有没有正确答案的来源，检索层的及格线。
2. **MRR**：正确答案排第几，越靠前越好，指导 k 值和 rerank 取舍。
3. **忠实度（Faithfulness）**：生成答案里的每个 claim 能不能在检索片段中找到依据，防幻觉的核心指标。
4. **答案相关性**：答的和问的是不是一回事，过滤"答非所问"。

- 小说引擎落地：20 个剧情问题（"林溯父母是谁""源石第一次出现在哪章"）做 golden 集，每次改 chunking/embedding 跑一遍回归，数字说话。
- 评测 skill 见本库第 3 条 `rag-eval-workflow`，它的分层定位（ingestion/chunking/retrieval/rerank/generation）就是按这套指标拆的。
