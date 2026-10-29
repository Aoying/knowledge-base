---
知识点: Git 运维与工具
层级: 基础
适用版本: 通用
更新日期: 2026-10-29
关联:
  - 工程实践知识点地图
  - Linux与Shell
标签:
  - 运维
  - 速查
  - 实践
---

# Git 运维与工具

> 每晚 20:30 自动补充完善。

## 常用命令速查

```bash
git clone <url>                 # 克隆
git status / git diff           # 看状态 / 看改动
git add . && git commit -m "x"  # 提交
git push / git pull --rebase    # 推 / 拉（rebase 保持历史线性）
git branch -a                   # 看所有分支
git checkout -b feat/x          # 新建并切换分支
git log --oneline -10           # 看最近 10 条提交
git stash / git stash pop       # 暂存 / 恢复现场
```

## 常见问题

### 1. 刚 commit 发现写错了，想撤回但保留改动
```bash
git reset --soft HEAD~1   # 退回提交，改动保留在暂存区
```
三档对比：`--soft`（改动留暂存区）/ `--mixed`（默认，改动回工作区）/ `--hard`（改动全丢，危险）。

### 2. 在错误分支上 commit 了，怎么搬到正确分支
```bash
git stash            # 在当前分支暂存
git checkout 正确分支
git stash pop        # 释放出来再 commit
```

### 3. pull/push 冲突
先 `git status` 看冲突文件，手动改完冲突标记（`<<<<<<<` 那几段），`git add` 后继续。原则：**先拉再推，推之前看一眼 diff**。

### 4. .gitignore 写了但不生效
文件已经被追踪了，ignore 对它无效：
```bash
git rm --cached <file>   # 取消追踪但保留本地文件
```

### 5. 中文文件名显示为 `\346\226\207` 乱码
```bash
git config --global core.quotepath false
```

### 6. Windows 换行符警告（LF/CRLF）
```bash
git config --global core.autocrlf true   # Windows 推荐：入库转 LF，检出转 CRLF
```

### 7. 想看某次提交改了什么
```bash
git show <commit-sha> --stat
```

### 8. 远端重组了目录，本地 pull 后旧文件夹还在

这是 Git 同步不会自动删除本地未跟踪文件夹导致的。进入仓库目录，执行：

```bash
git fetch origin
git status
git pull
git clean -nd
```

`git clean -nd` 只预览将删除的未跟踪文件，不会真的删除。

确认残余文件确实不需要后，再执行：

```bash
git clean -fd
```

注意：`git clean -fd` 会删除所有未跟踪文件和文件夹，请先检查预览结果。若残余文件已被 Git 跟踪但远程已删除，通常 `git pull` 就能同步删除。

### 9. commit 了但还没 push，想撤销

撤销未推送的 commit 是安全且推荐的，核心命令是 `git reset`，按想保留修改的程度选参数：

**保留修改，仅撤销 commit（推荐）**：
```bash
git reset --soft HEAD~1    # 修改保留在暂存区，可直接重新提交
git reset HEAD~1            # 默认 --mixed，修改回到未暂存状态，需重新 git add
```

**彻底丢弃修改（谨慎）**：
```bash
git reset --hard HEAD~1    # 永久删除工作区未提交改动，不可逆，执行前务必确认
```

补充技巧：撤销多个 commit 把 `HEAD~1` 换成 `HEAD~n`（如 `HEAD~3`）；只想改上一次的 commit message 用 `git commit --amend`，不需要 reset。

### 10. 本地仓库损坏，强行修复

`git fsck` 报大量 `error: object file ... is empty` 时，不想重新克隆可以尝试清理。**操作有风险，先备份整个项目文件夹。**

1. 找出损坏对象：
```bash
git fsck --full
```
2. 删除所有 0 字节空对象文件（在仓库根目录执行）：
```bash
find .git/objects -type f -empty -delete -print
```
3. 从远端重新获取完整数据：
```bash
git fetch --refetch
```
4. 还不行就重置：
```bash
git fetch origin
git reset --hard origin/你的分支名
```

额外建议：知识库推送体积大、网络不稳定导致 unexpected disconnect 时，增大缓冲区：
```bash
git config --global http.postBuffer 157286400
```
以上都无效，最稳妥的还是重新克隆。

### 11. push 被拒绝：non-fast-forward
- **现象**：`git push` 报 `! [rejected] main -> main (non-fast-forward)`。
- **原因**：远端有你本地没有的提交（比如别人推了，或你在别的机器推过）。
- **解决**：先 `git pull --rebase` 把远端提交接进来再推；确认远端提交是你自己的、不要了，才用 `git push --force-with-lease`（别用裸 `--force`）。

### 12. 大文件 push 一直失败
- **现象**：push 到 99% 断开，或远端拒绝超大文件。
- **原因**：GitHub 单文件 100MB 上限；大文件进仓库后每次 clone 都慢。
- **解决**：用 Git LFS（`git lfs track "*.psd"`）；已经误提交的大文件用 `git filter-repo` 清理历史；能放对象存储就别进仓库。

### 13. GitHub CLI（gh）常用

```bash
gh repo clone aoying0619/ob_note   # 不用拼 URL
gh pr create --fill                # 当前分支一键提 PR
gh pr list / gh pr view 123
gh pr checkout 123                 # 把别人的 PR 拉到本地看
gh issue create --title "xxx" --body "yyy"
gh workflow run nightly.yml        # 手动触发 Actions
gh api repos/{owner}/{repo}/commits --paginate  # 直接调 API
```

- `gh auth login` 一次登录，后面全自动，比在网页上点来点去快得多。
- 写脚本批量管仓库（如 50 个仓库统一改 branch 保护）就靠它。

### 14. rebase 工作流 + commit 规范

```bash
git checkout -b feat/xxx
# ...开发...
git fetch origin && git rebase origin/main   # 变基，保持线性历史
git push --force-with-lease                  # 自己分支才 force
# 合并：PR 用 Squash merge，一个功能一个 commit
```

- **commit 规范**（Conventional Commits）：`feat: xxx` / `fix: xxx` / `docs: xxx` / `refactor: xxx`，`git log` 一眼看懂。
- 个人分支用 rebase（历史干净），公共分支（main）永远 merge（别改写别人历史）。
- rebase 冲突 mid-way 想放弃：`git rebase --abort`，毫发无损。

### 15. 大文件处理（Git LFS 完整流程）

```bash
git lfs install
git lfs track "*.psd" "*.zip"     # 写 .gitattributes
git add .gitattributes
git add big.psd && git commit -m "feat: 素材"
git lfs ls-files                  # 确认进了 LFS
```

- FAQ 12 是"救火"，这是"日常"：仓库建好第一天就把二进制后缀 track 上。
- clone 时 `--depth 1` + LFS 拉取依然慢，大视频/数据集别进 git，用对象存储 + 版本号文件。
- `.gitattributes` 必须提交，不然别人 clone 下来的是指针文件。

### 16. 子模块 vs subtree

```bash
git submodule add https://github.com/x/y lib/y   # 子模块：仓库嵌仓库
git subtree add --prefix=lib/y https://github.com/x/y main  # subtree：合并成一个历史
```

|  | submodule | subtree |
|---|---|---|
| 历史 | 独立，父仓库只记 commit id | 合并进父仓库历史 |
| clone | 要 `--recurse-submodules` | 普通 clone 就行 |
| 适合 | 第三方库、主题 | 偶尔同步的小模块 |

- 个人知识库引第三方模板用 subtree，省心；团队协作的公共库用 submodule，版本锁定清楚。

### 17. reflog 救命三件套

```bash
git reflog                  # 所有操作历史，reset --hard 救回就靠它
git reset --hard HEAD@{2}   # 回到 2 个操作前
git fsck --lost-found       # 找悬空 commit（删分支误删的）
```

- `reset --hard` 丢了代码别慌，reflog 里 90 天内都找得回来。
- 分支删错了：`git branch <name> <reflog里的sha>` 原地复活。
- 终极：reflog 也清了才算真没了，所以别手贱 `git reflog expire`。

## Gitee / GitHub 对应

- 两个平台的 git 协议操作完全一样，区别只在远端地址和网页功能。
- 你的 ob_note（Gitee）和 knowledge-base（GitHub）都是私有仓，远端地址别贴到公开地方。


## 基础知识补强

### 核心概念与术语

- **Git 是什么**：分布式版本控制系统。分布式的意思是每个克隆下来的仓库都带完整历史，没有网络也能查看记录、提交代码；GitHub、Gitee 只是托管仓库的平台，不是 Git 本身。
- **快照模型**：Git 记录的是每次提交时整个项目的文件快照，不是逐行差异。没变的文件只存引用不重复占空间。理解这一点，很多命令的行为就顺理成章了。
- **三个区域**（新手必须先建立这个模型）：
  - **工作区**：你实际编辑文件的目录，改了还没 `add` 的都在这里。
  - **暂存区**：`git add` 把改动放进来的待提交清单，决定下一次提交到底包含哪些改动。
  - **本地仓库**：`git commit` 把暂存区内容固化成一次提交，存进 `.git` 目录。
- **`.git` 目录**：仓库的本体，历史、配置、对象全在里面。项目目录删掉 `.git` 就退化成普通文件夹，历史全没；反之只拷 `.git` 也能恢复出完整项目。
- **提交（commit）**：一次带说明的快照，有唯一哈希值（那串 40 位十六进制，平时用前 7 位就够）。每个提交指向它的父提交，串成一条历史链。
- **HEAD**：指向「你现在所在位置」的指针，通常指当前分支的最新提交。切分支、回退，本质都是在移动 HEAD。
- **分支（branch）**：本质只是一个指向某个提交的轻量指针，建分支几乎不花成本。在分支上提交，指针就往前挪；分支名不是文件夹，也不是代码副本。
- **远程与跟踪分支**：`origin` 是克隆时默认给远程仓库起的名字。`origin/main` 是远程分支在本地的镜像记录，你 `fetch` 时更新它；本地 `main` 和 `origin/main` 是两个不同的指针。
- **标签（tag）**：指向某个提交的固定标记，给版本留锚点，如 `v1.0`。和分支的区别是它不会随新提交移动。

### 基础语法与用法

第一次使用先做身份配置（提交记录里显示谁干的）：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
git config --list            # 查看当前生效的配置
```

从零建仓库和一次完整提交流程：

```bash
git init                     # 在当前目录建仓库（生成 .git 目录）
git status                   # 看每个文件处于什么状态，先养成习惯
git add 文件名                # 指定文件进暂存区；git add . 是全部
git diff --staged            # 提交前最后确认：暂存区里到底是什么
git commit -m "说明"         # 把暂存区固化成一次提交
git log --oneline --graph    # 一行一个提交地看历史和分支走向
```

文件在三个区域间的状态流转，对着记：

```bash
# 未跟踪(untracked)：新文件，Git 还不认识它，add 后变已暂存
# 已修改(modified)：改了被跟踪的文件，还没 add
# 已暂存(staged)：add 过了，等 commit
# 已提交(committed)：进入历史，工作区干净时 status 显示 nothing to commit
git diff          # 工作区 vs 暂存区：还没 add 的改动
git diff --staged # 暂存区 vs 上次提交：即将提交的改动
```

分支与合并的最基础形态（先懂 merge，再谈 rebase）：

```bash
git switch -c feat/login     # 新建并切到分支（老写法 checkout -b 等价）
git switch main              # 切回主分支
git merge feat/login         # 把 feat/login 的提交并进当前分支
git branch -d feat/login     # 合并完删掉分支指针（提交不会丢）
```

- merge 产生分叉汇合的历史，必要时会生成一个「合并提交」；特点是历史真实、不用改写已有提交，公共分支优先用它。
- 发生冲突不是操作失败，是 Git 把两边都改了同一处、无法自动决定的地方标出来等人裁决，属于正常流程。

远程同步的三个动作要分清：

```bash
git fetch origin             # 只把远程的最新状态拉到 origin/* 指针，不动你的代码
git pull                     # fetch + merge（或 rebase，取决于配置），一步到位
git push                     # 把本地提交推到远程；新分支第一次要带 -u 建跟踪关系
git push -u origin feat/login
```

- 不确定远程有什么变化时，先 `fetch` 再 `git log origin/main` 看一眼，最后决定怎么合，比直接 pull 稳。
- `push -u` 之后，这个分支以后直接 `git push`、`git pull` 就知道对应远程哪个分支。

### 关键机制速览

- **撤销操作的对应表**（先判断改动在哪个区域，再选命令）：
  - 改坏了工作区文件、还没 add：`git checkout -- 文件` 或 `git restore 文件`，丢弃这次修改。
  - add 错了：`git restore --staged 文件` 把它移出暂存区，文件内容不变。
  - 已 commit 没 push：用 reset 按保留程度回退（原文已有三档对比）。
  - 已 push：别改写历史，用 `git revert 提交` 生成一个反向提交来抵消，别人拉下来不冲突。
- **detached HEAD（游离状态）**：直接 `git checkout 某个提交` 时 HEAD 不挂在分支上。这时做的提交没有分支指针接着，切走就容易找不到。要保留成果先在当前位置建分支再继续。
- **合并冲突的标记含义**：冲突段里 `<<<<<<< HEAD` 到 `=======` 之间是你当前分支的内容，`=======` 到 `>>>>>>>` 之间是合进来的内容。改成最终想要的样子、删掉全部标记行，再 add 才算解决完。
- **.gitignore 的定位**：只对未被跟踪的文件生效，写的是路径匹配规则，一行一个。它解决「哪些文件不该进仓库」（编译产物、日志、本地配置），不是删除工具。
- **为什么强调先拉再推**：远程是共享状态，别人先推了提交，你本地历史就和远程分叉，push 会被拒绝。先把远程提交合进来再推，等于排队而不是抢道。

### 新手易错点

- **在错误的分支上工作而不自知**：动手前先 `git status` 看第一行写的是哪个分支。养成习惯比事后搬运提交省事得多。
- **`git add .` 不看内容就提交**：临时文件、调试代码、带敏感信息的本地配置会一起进暂存区。提交前用 `git diff --staged` 过一眼；敏感信息一旦进历史，删文件也删不掉记录，要改写历史才能清。
- **把 pull 当万能同步**：工作区有未提交改动时直接 pull，改动和拉下来的内容撞车会更乱。先 commit 或 stash 保存现场，再拉。
- **`git reset --hard` 当撤销万能药**：它会永久丢弃工作区和暂存区里未提交的改动，不可逆。新手阶段凡是要用 `--hard`，先停三秒确认改动真的不要了，或先随便 commit 一份保底。
- **对公共分支用 force push**：`push --force` 会覆盖远程历史，队友基于旧历史的工作直接作废。自己的特性分支要改写历史也优先 `--force-with-lease`，公共分支禁止。
- **提交说明写成「update」「fix」**：三个月后没人（包括你自己）知道这次提交干了什么。说明至少写清改了什么、为什么改，团队项目按约定的前缀规范写。
- **大文件或构建产物进仓库**：Git 适合管文本源码。jar、zip、视频进库后每次克隆都背着全部历史版本，仓库只会越来越臃肿，该进忽略清单或走专门的大文件方案。
