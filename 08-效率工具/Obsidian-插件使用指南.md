# Obsidian 插件使用指南

> 根据本仓库 `.obsidian/community-plugins.json` 生成（2026-10-17，共 21 个插件）。
> 插件有增减时由夜间维护任务同步更新本指南。

## 目录

- [obsidian-git](#1-obsidian-git)：自动 Git 备份
- [excalidraw](#2-excalidraw)：手绘白板
- [templater](#3-templater)：模板引擎
- [dataview](#4-dataview)：笔记查询
- [calendar](#5-calendar)：日历与日记
- [style-settings](#6-style-settings)：主题样式微调
- [copilot](#7-copilot)：AI 助手
- [quickadd](#8-quickadd)：快速捕获
- [editing-toolbar](#9-editing-toolbar)：编辑工具栏
- [icon-folder](#10-icon-folder)：文件夹图标
- [minimal-settings](#11-minimal-settings)：Minimal 主题设置
- [table-editor-obsidian](#12-table-editor-obsidian)：表格增强（Advanced Tables）
- [tasks](#13-tasks)：任务管理
- [kanban](#14-kanban)：看板
- [omnisearch](#15-omnisearch)：全文搜索增强
- [recent-files](#16-recent-files)：最近文件
- [brat](#17-brat)：安装测试版插件
- [multi-column-markdown](#18-multi-column-markdown)：多栏布局
- [file-cleaner-redux](#19-file-cleaner-redux)：清理无用附件
- [commander](#20-commander)：自定义命令栏
- [auto-link-title](#21-auto-link-title)：粘贴链接自动取标题
- [periodic-notes](#22-periodic-notes)：周期笔记

---

## 1. obsidian-git

**干嘛的**：把笔记库变成 Git 仓库，自动提交 + 推送，你的 vault backup 就是它干的。

**核心用法**：
- `Ctrl/Cmd + P` 打开命令面板，搜 "Obsidian Git"：`Commit all changes`（提交）、`Push`（推送）、`Pull`（拉取）
- 设置 → 打开 `Commit on file change` / `Auto pull` / `Auto push`，设间隔（如 10 分钟），实现全自动备份
- 右上角状态栏会显示同步状态（↑待推送 / ↓待拉取 / ✓已同步）

**技巧**：`.gitignore` 里忽略 `workspace.json` 这类个人布局文件，多端同步不打架。

## 2. excalidraw

**干嘛的**：手绘风白板画图，画架构图、流程图、草图，图可以直接 `[[嵌入]]` 到笔记里。

**核心用法**：
- 新建笔记时选 Excalidraw 类型，或命令面板 `Create new Excalidraw drawing`
- 左侧图形库拖拽：矩形、箭头、文字；`-->` 输入会自动变箭头
- 选中图形按 `Ctrl/Cmd + D` 复制样式；`Alt + 拖拽` 快速复制
- 笔记里用 `![[画图.excalidraw]]` 嵌入，可指定只显示局部区域

**技巧**：画 Flowable 流程、系统架构时比截图直观，且是矢量可改。

## 3. templater

**干嘛的**：模板引擎，比 Obsidian 自带模板强大得多（支持变量、日期运算、脚本）。

**核心用法**：
- 设置里指定模板文件夹（如 `模板/`），新建笔记时用 `Templater: Insert template` 插入
- 常用语法：`<% tp.date.now("YYYY-MM-DD") %>` 插入今天日期；`<% tp.file.title %>` 插入文件名
- `<% tp.file.creation_date() %>` 自动记录创建时间

**技巧**：给日记、周报、技术笔记各做一个模板，新建自动套用，省掉重复排版。

## 4. dataview

**干嘛的**：把笔记当数据库查，自动生成索引、表格、任务列表。

**核心用法**：
- 代码块语言选 `dataview`，写查询：
  ```dataview
  TABLE file.mtime as 修改时间
  FROM "java"
  SORT file.mtime DESC
  LIMIT 10
  ```
- `LIST` 列清单，`TASK` 聚合所有未完成任务：`TASK WHERE !completed`
- `FROM` 后跟文件夹、标签（`#tag`）或双向链接（`[[笔记]]`）

**技巧**：在 MOC（地图）笔记里放几个 dataview 查询，目录自动更新，不用手工维护索引。

## 5. calendar

**干嘛的**：侧边栏日历，点击日期直接打开/创建日记。

**核心用法**：
- 设置里绑定日记文件夹和日期格式（如 `日记/YYYY-MM-DD`）
- 点日期建日记；有内容的日期会显示圆点
- 配合 Periodic Notes 可扩展周记、月记

**技巧**：每天点一下日历写 3 行今日记录，坚持下来就是个人知识的时间线。

## 6. style-settings

**干嘛的**：可视化微调主题样式（字体、行距、颜色），不用写 CSS。

**核心用法**：
- 设置 → Style Settings，按分类调：字体大小、行高、编辑区宽度、标题颜色等
- 所有修改实时预览，关掉即恢复默认

**技巧**：只在你用的主题下生效；换主题后要重新调一遍。

## 7. copilot

**干嘛的**：在 Obsidian 里直接跟 AI 聊天，可引用当前笔记做问答。

**核心用法**：
- 侧边栏打开 Copilot 聊天窗口，选中笔记文字后点"引用"再提问
- 命令面板：`Copilot: Summarize` 总结当前笔记、`Copilot: Explain` 解释选中内容
- 设置里配自己的 API key 和模型

**技巧**：读官方文档时，选中一段直接问"这段话什么意思"，比切出去查快。

## 8. quickadd

**干嘛的**：一键快速捕获想法、建笔记、执行小宏，替代"我先记下来回头整理"。

**核心用法**：
- 设置里添加 QuickAdd：选类型 `Capture`（追加到指定笔记末尾）或 `Template`（用模板建新笔记）
- 给每个 QuickAdd 绑快捷键，如 `Ctrl/Cmd + Shift + J` 速记一条灵感到 `杂记.md`
- `Macro` 类型可串多个动作：建笔记 → 套模板 → 打开

**技巧**：灵感类用 Capture 追加，项目类用 Template 建独立笔记，入口统一走快捷键。

## 9. editing-toolbar

**干嘛的**：编辑器下方常驻格式化工具栏，手机端尤其好用（不用记 Markdown 符号）。

**核心用法**：
- 加粗、标题、列表、待办、代码块、表格、链接，一键插入
- 设置里可自定义按钮顺序和分组

**技巧**：手机上写笔记的主力，配合 QuickAdd 捕获，移动端体验拉满。

## 10. icon-folder

**干嘛的**：给文件夹和笔记换图标，目录一眼可辨。

**核心用法**：
- 右键文件夹 → `Change icon`，搜图标（如 java 选 ☕）
- 图标在文件树和链接处都会显示

**技巧**：只给一级目录加图标，加多了反而乱。

## 11. minimal-settings

**干嘛的**：Minimal 主题的专属设置面板（前提是你用的是 Minimal 主题）。

**核心用法**：
- 调整 Minimal 主题的各种细节：标题样式、列表缩进、引用样式、暗色模式配色
- 和 style-settings 配合，一个管主题专属项，一个管通用项

**技巧**：如果你换了别的主题，这个插件就没用了，可以直接禁用。

---

## 12. table-editor-obsidian（Advanced Tables）

**干嘛的**：表格增强，写 Markdown 表格不再痛苦。

**核心用法**：
- 表格里按 `Tab` 跳到下一单元格、`Enter` 换行，自动对齐竖线
- 命令面板：插入/删除行列、移动行列、按列排序、行列转置
- 支持表格公式（如 `=sum(@row)`），命令 `Evaluate formulas` 计算

**技巧**：整理快捷键对照表、对比表时用它，排版一次成型。

## 13. obsidian-tasks-plugin（Tasks）

**干嘛的**：任务管理，给 `- [ ]` 待办加上日期、优先级、循环，然后跨笔记聚合查询。

**核心用法**：
- 待办写法：`- [ ] 写周报 📅 2026-10-20 ⏫`（📅 截止日期、⏫ 高优先级、🔁 循环）
- 代码块语言选 `tasks`，写查询聚合所有笔记的任务：
  ```tasks
  not done
  sort by due
  ```
- 命令面板 `Tasks: Toggle task done` 快速勾选

**技巧**：日记里随手记 `- [ ]`，再建一个"任务中心"笔记用 tasks 查询全部收拢，按截止日期排序。

## 14. obsidian-kanban

**干嘛的**：看板，拖拽卡片管理任务。

**核心用法**：
- 新建 Kanban 类型笔记，列如"待办 / 进行中 / 完成"，卡片拖拽流转
- 卡片可以是纯文本，也可以 `[[链接]]` 到某篇笔记
- 右键卡片可设日期、标签

**技巧**：适合管项目阶段（如小说引擎的"待开发/开发中/已完成"），比纯待办列表直观。

## 15. omnisearch

**干嘛的**：全文搜索增强，比 Obsidian 自带搜索更快更准。

**核心用法**：
- `Ctrl/Cmd + Shift + F` 唤起，按关键词模糊搜全文
- 支持 PDF 附件内容搜索、图片 OCR 文字搜索
- 搜索结果按相关度排序，支持排除路径

**技巧**：知识库大了以后找东西的主力入口；配合文件命名规范，基本不用翻文件夹。

## 16. recent-files-obsidian（Recent Files）

**干嘛的**：最近打开的文件列表，快速切回。

**核心用法**：
- 侧边栏或命令面板打开最近文件列表，一键跳转
- 设置里可调显示数量、是否排除某些路径

**技巧**：和 Omnisearch 配合：一个管"找得到"，一个管"回得去"。

## 17. obsidian42-brat（BRAT）

**干嘛的**：从 GitHub 直接安装测试版/未上架的插件。

**核心用法**：
- 设置 → BRAT → Add Beta plugin，填 GitHub 仓库地址（如 `作者/插件名`）
- 可一键检查所有 beta 插件的更新

**技巧**：只用来装你信任的作者的测试版；正式版出了就换回社区版，beta 版可能有 bug。

## 18. multi-column-markdown

**干嘛的**：多栏布局，并排显示内容。

**核心用法**：
- 用特定语法把内容分成左右两栏（看插件说明的 callout 写法）
- 适合做对比：左中文右英文、左方案 A 右方案 B

**技巧**：写对比类笔记（RocketMQ vs Kafka 选型）时用，比上下堆表格直观。

## 19. file-cleaner-redux（File Cleaner Redux）

**干嘛的**：扫描并清理没有被任何笔记引用的附件（图片、PDF）。

**核心用法**：
- 命令面板 `File Cleaner Redux: Clean files`，先预览列表再确认删除
- 设置里可指定附件文件夹、排除规则

**技巧**：定期跑一次，知识库附件只会越积越多；删除前看一眼预览，防止误删。

## 20. cmdr（Commander）

**干嘛的**：自定义命令栏，把常用命令放到顺手的位置。

**核心用法**：
- 设置里把任意命令添加到：左侧 ribbon 图标栏、右键菜单、手机端工具栏
- 比如把"QuickAdd 速记""Omnisearch""Git 推送"做成一键按钮

**技巧**：手机端尤其有用——配合 editing-toolbar，常用操作都不用进命令面板翻。

## 21. obsidian-auto-link-title

**干嘛的**：粘贴链接时自动抓取网页标题，不用手动写 `[标题](链接)`。

**核心用法**：
- 直接粘贴 URL，自动变成 `[网页标题](链接)`
- 抓不到标题时回退为裸链接

**技巧**：收集技术博客、论文链接时省掉复制标题的步骤；内网或反爬页面可能抓不到，属正常。

## 22. periodic-notes（Periodic Notes）

**干嘛的**：周期笔记——日记/周记/月记/季记/年记，和 calendar、templater 配套用。

**核心用法**：
- 设置里为日/周/月记各绑一个模板和存放路径
- calendar 插件点日期时自动按这里的配置建笔记
- 命令面板 `Open daily note` 等快捷命令

**技巧**：你已经有 calendar + templater 了，三个配齐：calendar 负责点日期，periodic-notes 负责建对应周期笔记，templater 负责套模板。周记模板里可以放 dataview 查询，自动汇总本周任务。
