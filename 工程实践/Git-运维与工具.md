---
知识点: Git 运维与工具
层级: 基础
适用版本: 通用
更新日期: 2026-11-14
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

## 小技巧与冷知识

【冷知识】Git 的名字是 Linus Torvalds 的自嘲：2005 年他为 Linux 内核写这个版本控制系统时，把它取名叫 git——在英式俚语里 git 是“蠢货、不讨人喜欢的人”的意思。Linus 自己说过，他是个自大狂，所以所有项目都以自己命名，先是 Linux，现在是 git；名字本身就是一句玩笑，不是什么缩写。

【小技巧】想找“某一行代码是哪次提交改坏的”，别一页页翻 log，用 `git log -S "关键词" --oneline`：它只列出增删了这个字符串的提交，改动行数、重构提交都不会干扰结果。配合 `git show <提交>` 看那次到底改了什么，定位回归问题比二分猜测快得多；`-S` 搜的是字符串出现次数的变化，改标点、换行这类纯格式改动不会触发它。

【小技巧】回归 bug 不知道哪次提交引入的，用 `git bisect` 二分定位：先 `git bisect start`，在已知好的提交上 `git bisect good`、坏的提交上 `git bisect bad`，Git 会自动切到中间提交让你测，每次回 good 或 bad，几轮就锁定第一个坏提交。手动翻几十条提交是线性排查，二分是指数级收口；找出来后直接 `git bisect reset` 回到原来分支。

【小技巧】两个分支要来回对照、又不想反复 stash 和切换，用 `git worktree`：`git worktree add ../项目名-分支名 分支名` 会在另一个目录检出同一仓库的另一个分支，两个目录共用一份 `.git` 历史、各自独立工作。比 clone 两份省磁盘、比来回 checkout 省现场，修热修复分支时主分支的进度一点不受影响；用完 `git worktree remove` 收掉。

【小技巧】一次改动里混了两件事，别整包提交，用 `git add -p` 交互式分块：它按代码块逐个问你要不要暂存，y 暂存、n 跳过、s 再拆细，一次提交只进一件事。提交粒度干净了，之后 `git log`、`git revert`、`git bisect` 全部受益；原文只讲了 `git add .` 全量提交，分块提交是把“一个提交只干一件事”落到操作层的做法。

【小技巧】stash 默认不带未跟踪文件，换分支前先确认现场收干净了：`git stash push -u`（或 `--include-untracked`）才把新文件也一起暂存，否则切分支后新文件会跟着跑过去。恢复时 `git stash list` 看编号、`git stash apply stash@{1}` 指定恢复哪一份，比只会 `stash pop` 恢复最新一份更稳；pop 失败时暂存还在，apply 失败同样不丢，确认无误再 `git stash drop`。

【冷知识】新仓库默认分支是 main 还是 master，取决于“谁建的库”：GitHub 从 2020 年 10 月起新建仓库默认分支改为 main，但本地 `git init` 的默认分支仍是 master。想让本地也默认 main，配一次就行：`git config --global init.defaultBranch main`（Git 2.28 及以上版本支持）；已建好的仓库改默认分支要去远端设置里改，本地只改名不同步。

【小技巧】把常用长命令收进别名，少敲一半字：`git config --global alias.lg "log --oneline --graph --all --decorate"` 之后 `git lg` 就是全部分支带图的历史；`alias.st "status -sb"` 看分支加状态、`alias.last "log -1 --stat"` 看上一次提交改了什么。别名存在 `~/.gitconfig` 里，换机器拷走这一个文件就能复用，比每次翻速查表快。

【冷知识】Git 命令分两层：日常用的 status、add、commit、log 叫“瓷器”（porcelain），是给人用的友好接口；底下还有一套“管道”（plumbing）命令，如 `git cat-file`、`git hash-object`、`git update-index`，做的是读写对象、算哈希这类原子操作。排障时瓷器命令不够用就下沉到管道层——比如 `git cat-file -p HEAD` 直接看某个提交对象的内容，绕过所有友好包装看到原始数据。

【小技巧】提交前跑一遍 `git diff --check`，能拦住最常见的低级错误：它会标出新增行尾的空白、行首混用的空格与制表符，这类问题进了仓库会在 diff 里一直飘红、还容易在合并时制造假冲突。把它放进提交前的固定动作，比事后在评审里被人指出要省事；团队项目里也可以配成 pre-commit 钩子自动跑。

## 原理剖析

本节只综合原文已有章节的结论：对象与引用的说法来自“基础知识补强”，压缩与体量的数字来自“对象存储与传输定量”，撤销、冲突与推送的规则来自“常见问题”。不引入原文之外的数字与命令行为；凡标“本库实测”的是 2026-11-10 在本知识库公开克隆上的实测值，凡标“Git 通用口径”的是原文定量节第 5 节已按 Git 官方文档核验过的说法，其余为原文经验口径。

### 对象模型与内容寻址

Git 存的不是逐行差异，而是一次提交时整个项目的文件快照（Git 通用口径；“基础知识补强”）。没变的文件只存引用、不重复占空间，这一点决定了后面很多命令的行为。

对象只有三类，各司其职：

- **blob**：一个版本的文件内容，不含文件名，只含内容本身。
- **tree**：一层目录的清单，记文件名、权限与它指向的 blob 或子 tree，拼起来就是某次提交的整棵目录树。
- **commit**：一次提交的记录，指向一棵 tree、指向父提交，并带提交说明与作者等信息。每个提交指向它的父提交，串成一条历史链。

寻址靠内容本身，不靠文件名或存放位置：每个对象入库前先拼一个对象头再算哈希——以 blob 为例，头是 `blob <字节数>\0` 再加文件内容，commit 与 tree 同理、只是类型名不同（Git 通用口径）。SHA-1 哈希为 20 字节，写作 40 位十六进制；SHA-256 为 32 字节，写作 64 位十六进制；本库用 SHA-1（Git 通用口径）。内容相同则哈希相同，内容一变则哈希全变，所以同一个文件、同一个版本在库里只存一份，改动过的版本才另存一份。

本库实测可以直观看到这个模型：全量克隆对象总数 4331 个（blob 1655 + commit 730 + tree 1946），对象未压缩总大小约 43.73 MiB（blob 42.57 + tree 0.92 + commit 0.23；三个分量均为两位小数修约值，直接相加为 43.72 MiB，与 43.73 MiB 的 0.01 MiB 之差来自修约）。也就是说，730 次提交的历史在对象层面就是 730 个 commit 对象加它们指向的 tree 与 blob，而不是 730 份完整代码拷贝。

### 引用、HEAD 与 reflog

分支、标签、远程跟踪分支都不是代码副本，只是引用——指向某个提交的指针：

- **分支（branch）**：一个指向某个提交的轻量指针，建分支几乎不花成本。在分支上提交，指针就往前挪；分支名不是文件夹，也不是代码副本。
- **标签（tag）**：指向某个提交的固定标记，给版本留锚点，如 `v1.0`。和分支的区别是它不会随新提交移动。
- **HEAD**：指向“你现在所在位置”的指针，通常指当前分支的最新提交。切分支、回退，本质都是在移动 HEAD。
- **远程与跟踪分支**：`origin` 是克隆时默认给远程仓库起的名字。`origin/main` 是远程分支在本地的镜像记录，`fetch` 时更新它；本地 `main` 和 `origin/main` 是两个不同的指针。

引用轻量，所以“删分支”删的只是指针，不删它指向的提交对象本身——提交对象还在不在，取决于还有没有别的引用或 reflog 指着它。`git reflog` 记的是 HEAD 与引用每一次移动的操作历史，`reset --hard` 丢了代码、误删了分支，都先从这里找（原文经验口径：reflog 里 90 天内都找得回来；只有 reflog 也清了才算真没了，所以别手贱 `git reflog expire`）。`detached HEAD`（游离状态）是同一机制的反面：直接检出某个提交时 HEAD 不挂在分支上，这时做的提交没有分支指针接着，切走就容易找不到，要保留成果先在当前位置建分支再继续。

### packfile 与 delta 压缩

对象有两种存放形态。刚提交的对象是松散对象：单个对象以 zlib 压缩后按对象 ID 前两位分目录存放在 `.git/objects/` 下；打包后则进 packfile（Git 通用口径）。压缩来自两层机制：松散对象的 zlib 压缩，以及 packfile 的 delta 压缩。

delta 压缩的思路是：packfile 中相似版本的对象只存一份完整内容，其余存与基准对象的差值（delta）；一条 delta 链可以多层嵌套，链越长、解包时要逐层还原的次数越多（Git 通用口径）。本库实测最长 delta 链为 14（`git verify-pack`，2026-11-10）。

本库实测能算清这两层压缩合起来省了多少：对象未压缩总大小约 43.73 MiB，pack 单文件 14,004 KiB（约 13.68 MiB），压缩比 = 43.73 ÷ 13.68 ≈ 3.2，省掉约 30 MiB（43.73 − 13.68 = 30.05 MiB）。但这个 3.2 倍是本库实测值，不是 Git 的固定值，边界条件原文定量节第 7 节已写明：文本 md 压得动，二进制与已压缩内容几乎压不动；delta 只对相似版本有效，内容完全不同的两个版本之间做不出有效的 delta，只能各存一份。本库就是例证：最大的 5 个 blob 全是 `.obsidian` 插件文件，合计约 17.09 MiB，占全部 blob 原始字节的约 40%，约为全部 268 篇笔记正文（约 4.07 MiB）的 4.2 倍；当前树压缩比 32.47 ÷ 11.92 ≈ 2.72，低于全库平均 3.2，正是因为当前树 87.5% 是难压缩的 `.obsidian` JS/CSS（28.40 MiB），而历史版本多为易压缩的 md 文本，拉高了整体压缩比。

### 提交图：merge、rebase 与 fast-forward

提交按父子关系连成一张图，merge 与 rebase 是对这张图做的两种不同操作：

- **merge**：把另一条分支的提交并进当前分支，产生分叉汇合的历史，必要时会生成一个“合并提交”；特点是历史真实、不用改写已有提交，公共分支优先用它。发生冲突不是操作失败，是 Git 把两边都改了同一处、无法自动决定的地方标出来等人裁决，属于正常流程。
- **rebase**：把当前分支的提交移到新的基底上重放，得到一条线性历史，不产生合并提交。个人分支用 rebase（历史干净），公共分支（main）永远 merge（别改写别人历史）；rebase 冲突中途想放弃，用 `git rebase --abort`，毫发无损。

**fast-forward（快进）**是 merge 与 push 共用的一条判断：只有当当前分支的尖端是目标提交的祖先、历史没有分叉时，指针才能直接往前挪，不用生成合并提交；一旦两边各有对方没有的提交，历史就分叉了，快进不成立。`git push` 报 `non-fast-forward` 就是这个判断在推送侧的表现：远端有你本地没有的提交（比如别人推了，或你在别的机器推过），本地历史和远程分叉了。正确处理是先 `git pull --rebase` 把远端提交接进来再推；确认远端提交是你自己的、不要了，才用 `git push --force-with-lease`（别用裸 `--force`），公共分支更要禁止 force push。

### 工作区、暂存区与仓库：三区流转

三个区域是一条单向固化链，改动在哪个区域，决定了该用什么命令撤销：

- **工作区**：你实际编辑文件的目录，改了还没 `add` 的都在这里。
- **暂存区**：`git add` 把改动放进来的待提交清单，决定下一次提交到底包含哪些改动。
- **本地仓库**：`git commit` 把暂存区内容固化成一次提交，存进 `.git` 目录。

对照关系是：`git diff` 看工作区 vs 暂存区（还没 `add` 的改动），`git diff --staged` 看暂存区 vs 上次提交（即将提交的改动）。文件在三个区域间的状态也按此流转：未跟踪（untracked）是新文件，`add` 后变已暂存；已修改（modified）是改了被跟踪的文件、还没 `add`；已暂存（staged）是 `add` 过了、等 `commit`；已提交（committed）是进入历史，工作区干净时 `status` 显示 nothing to commit。

撤销要先判断改动在哪个区域，再选命令（“基础知识补强”关键机制速览与“常见问题”第 1、9 条的对应表）：改坏了工作区文件、还没 `add`，用 `git checkout -- 文件` 或 `git restore 文件` 丢弃这次修改；`add` 错了，用 `git restore --staged 文件` 把它移出暂存区，文件内容不变；已 commit 没 push，用 reset 按保留程度回退——`--soft`（改动留暂存区）、`--mixed`（默认，改动回工作区）、`--hard`（改动全丢，危险）；已 push，别改写历史，用 `git revert 提交` 生成一个反向提交来抵消，别人拉下来不冲突。`git reset --hard` 会永久丢弃工作区和暂存区里未提交的改动，不可逆，新手阶段凡是要用 `--hard`，先停三秒确认改动真的不要了，或先随便 commit 一份保底。

远程同步的三个动作也要按三区模型分清：`git fetch origin` 只把远程的最新状态拉到 `origin/*` 指针，不动你的代码；`git pull` 是 fetch + merge（或 rebase，取决于配置），一步到位；`git push` 是把本地提交推到远程。不确定远程有什么变化时，先 `fetch` 再看一眼远程指针，最后决定怎么合，比直接 pull 稳；工作区有未提交改动时直接 pull，改动和拉下来的内容撞车会更乱，先 commit 或 stash 保存现场，再拉。

最后回到存放本身：`.git` 目录是仓库的本体，历史、配置、对象全在里面。项目目录删掉 `.git` 就退化成普通文件夹，历史全没；反之只拷 `.git` 也能恢复出完整项目。本库实测也印证了 `.git` 与工作区是两种口径：`.git` 约 14M、工作区约 34M，前者是压缩后的对象库 + 索引，后者是解压后的全部文件（含 `.obsidian` 插件等），不能拿一方大小直接推断另一方。

## 对象存储与传输定量：对象、packfile 与 clone 体量算一算

> 本节所有“本库实测”数字均取自 2026-11-10 在本知识库公开克隆（`Aoying/knowledge-base`，`main` 分支）上的实测；通用原理部分标注为 Git 通用口径，来源为 Git 官方文档（核验日期 2026-11-10）。换算均用 Python 当场复算，公式与中间值写在文中。

### 1. 本库实测快照（2026-11-10）

| 指标 | 全量克隆 | 浅克隆 `--depth 1` |
| --- | --- | --- |
| 对象总数 | 4331 个（blob 1655 + commit 730 + tree 1946） | 452 个 |
| pack 单文件 | 14,004 KiB（约 13.68 MiB） | 12,211 KiB（约 11.92 MiB） |
| `.git` 目录 | 约 14M | 约 13M |
| 工作区（不含 `.git`） | 约 34M | — |

其他实测值：

- 268 篇 md 合计 4,270,805 字节 ≈ 4.07 MiB（4,270,805 ÷ 1024 ÷ 1024 = 4.073）。
- 对象未压缩总大小约 43.73 MiB（blob 42.57 + tree 0.92 + commit 0.23；三个分量均为两位小数修约值，直接相加为 43.72 MiB，与 43.73 MiB 的 0.01 MiB 之差来自修约）。
- 超过 100 KB 的 blob 共 28 个。
- `git verify-pack` 实测：本库最长 delta 链为 14。

### 2. 换算一：pack 压缩比约 3.2 倍

公式：压缩比 = 对象未压缩总大小 ÷ pack 大小。

- 43.73 MiB ÷ 13.68 MiB = 3.196… ≈ 3.2。
- 其中 13.68 MiB = 14,004 KiB ÷ 1024 = 13.676 MiB（修约后 13.68 MiB）。

也就是说，Git 把约 43.73 MiB 的对象内容压进了约 13.68 MiB 的 pack 文件，省掉约 30 MiB（43.73 − 13.68 = 30.05 MiB）。压缩来自两层机制（见第 5 节）：松散对象的 zlib 压缩，以及 packfile 的 delta 压缩。

### 3. 换算二：浅克隆到底省了什么

只省历史，不省最新版。公式：节省量 = 全量 pack − 浅克隆 pack。

- 对象数之差：4331 − 452 = 3879 个对象，对应 729 个旧提交（730 − 1）的增量历史。
- pack 大小之差：14,004 − 12,211 = 1,793 KiB = 1,793 ÷ 1024 = 1.75 MiB。
- 节省比例：1,793 ÷ 14,004 = 12.80% ≈ 13%。

结论：`--depth 1` 只比全量克隆省约 1.8 MiB（约 13%），因为 pack 里的大头是最新版 blob，而不是历史。729 个旧提交的增量历史一共只占约 1.75 MiB——本库的提交历史本身很“轻”，历史不是体积问题。

### 4. 换算三：体积大头是谁——不是笔记，是插件

最大的 5 个 blob 全是 `.obsidian` 插件文件（本库实测 2026-11-10）：

| 文件 | 字节数 | 换算 |
| --- | --- | --- |
| `.obsidian/plugins/excalidraw/main.js` | 5,061,558 | 约 4.83 MiB |
| `.obsidian/plugins/copilot/main.js` | 4,963,948 | 约 4.73 MiB |
| `.obsidian/plugins/remotely-save/main.js` | 4,249,085 | 约 4.05 MiB |
| `.obsidian/themes/Catppuccin/theme.css` | 2,342,309 | 约 2.23 MiB |
| `.obsidian/plugins/dataview/main.js` | 1,302,069 | 约 1.24 MiB |

合计：5,061,558 + 4,963,948 + 4,249,085 + 2,342,309 + 1,302,069 = 17,918,969 字节 ≈ 17.09 MiB（17,918,969 ÷ 1024 ÷ 1024 = 17.089）。

占比换算：

- 占全部 blob 原始字节：17.09 MiB ÷ 42.57 MiB = 40.1% ≈ 40%。
- 268 篇 md 只占：4.07 MiB ÷ 42.57 MiB = 9.6% ≈ 10%。

结论：5 个插件文件（约 17.09 MiB）≈ 4.2 倍于全部 268 篇笔记正文（约 4.07 MiB）。本库的体积大头是 `.obsidian` 插件的 JS/CSS，而不是知识内容本身；工作区约 34M 中 md 仅占约 4.07 MiB，差额主要来自 `.obsidian` 等非 md 文件。

### 5. 通用原理：对象、对象头与 packfile（Git 通用口径）

来源：Git 官方文档（gitformat-objects、gitformat-pack、git-clone 相关章节），核验日期 2026-11-10。

- **对象头格式**：每个 Git 对象在入库前都有一个头，以 blob 为例是 `blob <字节数>\0` 再加文件内容，即头 = 类型名 + 空格 + 内容字节数的十进制表示 + 一个 NUL 字节，之后才是原始内容。commit 与 tree 同理，只是类型名不同。
- **对象 ID 长度**：SHA-1 哈希为 20 字节，写作 40 位十六进制；SHA-256 为 32 字节，写作 64 位十六进制。本库用 SHA-1（40 位十六进制对象 ID）。
- **松散对象**：单个对象以 zlib 压缩后按对象 ID 前两位分目录存放在 `.git/objects/` 下；`git gc` / `git repack` 会把松散对象打包进 packfile。
- **packfile 与 delta 链**：packfile 中相似版本的对象只存一份完整内容，其余存与基准对象的差值（delta）；一条 delta 链可以多层嵌套，链越长、解包时要逐层还原的次数越多。本库实测最长 delta 链为 14（`git verify-pack`，2026-11-10）。

### 6. clone 体量速算：pack 大小 ≈ 最新版压缩后 + 历史增量

速算公式：

```text
pack 大小 ≈ （最新版文件原始大小 ÷ 当前树压缩比）+ 历史增量
```

代入本库实测值验算：

- 最新版文件原始大小 32.47 MiB（HEAD 树 381 个文件合计 34,047,963 字节）；当前树压缩比 = 32.47 ÷ 11.92 ≈ 2.72，低于全库平均 3.2——因为当前树 87.5% 是难压缩的 `.obsidian` JS/CSS（28.40 MiB），而历史版本多为易压缩的 md 文本，拉高了整体压缩比。
- 32.47 ÷ 2.72 ≈ 11.93 MiB，与浅克隆 pack 实测 11.92 MiB 基本吻合。
- 再加历史增量约 1.75 MiB：11.92 + 1.75 = 13.67 MiB ≈ 全量 pack 实测 13.68 MiB，分解自洽。

**估算练习：如果插件 `main.js` 再翻一倍，pack 大约涨多少？**

- 新增原始体积 = 现有 5 大文件合计 17.09 MiB（翻一倍即新增一份同等大小）。
- 已打包压缩过的 JS 再压不动：按当前树压缩比 2.72 折算约 +6.3 MiB（17.09 ÷ 2.72 = 6.28）；按全库平均 3.2 折算约 +5.3 MiB（17.09 ÷ 3.2 = 5.34），后者偏乐观。
- 对照：全量 pack 将从约 13.68 MiB 涨到约 19.0–20.0 MiB（13.68 + 5.34 = 19.02；13.68 + 6.28 = 19.96），涨幅约 39%–46%。可见只要再塞几个 MB 级的插件 JS，pack 体积就会显著膨胀；反过来，瘦身也应先从 `.obsidian` 插件文件入手，而不是删笔记。

### 7. 边界条件

1. **压缩比随内容而变**：3.2 倍是本库实测值，不是 Git 的固定值。文本 md 压得动，二进制与已压缩内容几乎压不动；换一个以图片、视频或编译产物为主的仓库，压缩比会低得多。
2. **二进制 JS 几乎压不动**：`.obsidian` 插件的 `main.js` 是已打包压缩过的 JS，zlib 再压也省不了多少；第 6 节按 3.2 折算已是偏乐观的估计，实际涨幅可能更大。
3. **delta 只对相似版本有效**：delta 压缩的前提是新旧版本内容高度相似（如逐版小改的 md 笔记）；内容完全不同的两个版本之间做不出有效的 delta，只能各存一份。
4. **浅克隆省的是对象数，不是最新版体积**：`--depth 1` 砍掉的是 3879 个历史对象（约 1.75 MiB），最新版的 452 个对象（约 11.92 MiB）一个都少不了；历史轻的仓库用浅克隆省不了多少。
5. **`.git` 大小 ≠ 工作区大小**：本库 `.git` 约 14M、工作区约 34M，两者统计口径不同——`.git` 是压缩后的对象库 + 索引，工作区是解压后的全部文件（含 `.obsidian` 插件等）；不能拿 `.git` 大小直接推断工作区大小，反之亦然。

## 实战案例

### 案例 1：误删分支后用 reflog 找回

以下为设计演练示例，非真实生产事故。

**现象**

分支上的提交已经做完、还没推到远端，清理分支时误删了这个分支指针。随后 `git branch -a` 里找不到该分支，`git log --oneline -10` 也看不到那几条提交，工作区是干净的、文件本身没有报错。

**排查**

先确认丢的是指针还是提交对象。`git status` 看当前分支与工作区状态，确认没有未提交改动干扰判断；再 `git reflog` 看 HEAD 与引用最近的操作历史，找删分支之前的那次提交记录——reflog 记的是操作历史，不是当前分支的提交历史，所以分支指针删了，操作记录还在。

**根因**

分支本质只是一个指向某个提交的轻量指针，删分支删的只是这个指针，不删它指向的提交对象本身。只要还有 reflog 指着那次提交，提交对象就还在库里；切走、回退、删分支这类操作移动或删掉的都是指针，对象本身不会立刻消失。反过来，如果连 reflog 也清了，才算真没了。

**修复**

在 reflog 里找到删分支前那次提交的 SHA，用原来的分支名在原地复活指针：

```bash
git reflog
git branch <name> <reflog里的sha>
```

如果丢的不只是分支指针、而是 `reset --hard` 退回后想回到退回前的状态，用 reflog 里的操作序号直接回去：

```bash
git reset --hard HEAD@{2}
```

还可以用 `git fsck --lost-found` 找悬空 commit（删分支误删的）。复活后先 `git log --oneline -10` 核对提交都在，再 `git status` 确认工作区干净，确认无误后再继续开发或推送。

**复盘**

- 删分支前先看一眼 `git log --oneline -10`，确认分支上的提交已经合并或已经推送，不要只凭分支名判断。
- `reset --hard` 丢了代码别慌，reflog 里 90 天内都找得回来（原文经验口径）；但不要养成依赖 reflog 的习惯，重要提交先推到远端再清理本地分支。
- 游离状态（detached HEAD）下做的提交没有分支指针接着，切走就容易找不到；要保留成果，先在当前位置建分支再继续。
- 切记不要用 `git reflog expire` 清 reflog，那是把最后一条找回的路也断掉。

### 案例 2：大文件误提交后仓库膨胀的清理

以下为设计演练示例，非真实生产事故。

**现象**

一次提交里误带了一个大文件，之后 `git push` 到 99% 断开，或远端直接拒绝超大文件；即使这次推上去了，以后每次 clone 都很慢，仓库越用越臃肿。

**排查**

先 `git status` 确认工作区干净、问题不在未提交的改动；再 `git log --oneline -10` 定位是哪次提交带进了大文件，用 `git show <commit-sha> --stat` 看那次提交到底改了什么、确认大文件的文件名与大小。只删工作区里的文件没用——文件已经进了提交历史，历史里的每一次提交都还背着它。

**根因**

两条原文规则叠加：一是 GitHub 单文件 100MB 上限，大文件进仓库后每次 clone 都慢；二是 Git 适合管文本源码，大文件或构建产物进库后，仓库要背着全部历史版本，只会越来越臃肿。`.gitignore` 也救不了这一次：它只对未被跟踪的文件生效，文件一旦被追踪、进了提交，ignore 对它无效；而且敏感信息一旦进历史，删文件也删不掉记录，要改写历史才能清。

**修复**

已经误提交的大文件用 `git filter-repo` 清理历史（“常见问题”第 12 条的救火办法）。清理完历史、确认本地提交记录里已经没有那个大文件后，再把大文件纳入日常管理，防止下一次又误提交：

```bash
git lfs install
git lfs track "*.psd"
git add .gitattributes
git add big.psd && git commit -m "feat: 素材"
git lfs ls-files
```

这里 `git lfs track "*.psd"` 是写 `.gitattributes`，`.gitattributes` 必须提交，不然别人 clone 下来的是指针文件。历史已改写，推送时按“常见问题”第 11、14 条的规则办：只有确认远端提交是你自己的、不要了，才用 `git push --force-with-lease`（别用裸 `--force`），自己的特性分支才 force，公共分支禁止 force push；推之前先 `git pull --rebase` 确认远端没有别人新推的提交，避免把别人的提交覆盖掉。推完后让协作者重新克隆或按新历史同步，不要在旧历史的基础上继续提交。

**复盘**

- FAQ 12 是“救火”，FAQ 15 是“日常”：仓库建好第一天就把二进制后缀 track 上，别等误提交了再清理。
- clone 时 `--depth 1` + LFS 拉取依然慢，大视频、数据集别进 git，用对象存储 + 版本号文件；能放对象存储就别进仓库。
- 提交前先 `git status` 看一眼状态、再 `git diff` 看一眼改动，别 `git add .` 不看内容就提交；临时文件、调试代码、带敏感信息的本地配置会一起进暂存区。
- 体量判断上也要分清主次：本库实测里体积大头是 `.obsidian` 插件文件而不是笔记正文，瘦身先从大文件、插件文件这类大 blob 入手，而不是删笔记；浅克隆只省历史、不省最新版，历史轻的仓库用浅克隆省不了多少。

## 面试与自测

1. `git reset` 的 `--soft`、`--mixed`、`--hard` 三档各保留什么？已经 push 的提交想撤销，为什么不能用 reset？
2. `.gitignore` 写了规则但不生效，最常见的原因是什么？怎么处理？
3. `push` 被拒绝、报 `non-fast-forward` 的原因是什么？正确的处理步骤是什么，什么情况下才用 `--force-with-lease`？
4. 为什么对本库用 `--depth 1` 浅克隆只省了约 13%？pack 里的大头是什么？

### 参考答案

1. `--soft` 是退回提交、改动保留在暂存区；`--mixed` 是默认值，改动回到工作区；`--hard` 是改动全丢且不可逆，执行前务必确认。已经 push 的提交不能用 reset 改写历史，否则别人拉下来会冲突，应该用 `git revert 提交` 生成一个反向提交来抵消。
2. 最常见的原因是文件已经被追踪了，ignore 只对未被跟踪的文件生效。处理办法是 `git rm --cached <file>`，取消追踪但保留本地文件；注意 ignore 解决的是“哪些文件不该进仓库”，不是删除工具。
3. 原因是远端有你本地没有的提交，比如别人在别的机器上推过，本地历史和远程分叉了。正确处理是先 `git pull --rebase` 把远端提交接进来再推；只有确认远端提交是你自己的、不要了，才用 `git push --force-with-lease`，别用裸 `--force`，公共分支更要禁止 force push。
4. 因为 pack 里的大头是最新版 blob：浅克隆 pack 约 11.92 MiB，全量 pack 约 13.68 MiB，729 个旧提交的历史增量一共只占约 1.75 MiB，省掉历史只省约 13%。体积大头是 `.obsidian` 插件文件：最大的 5 个 blob（excalidraw、copilot、remotely-save 的 main.js、Catppuccin theme.css、dataview main.js）合计约 17.09 MiB，占全部 blob 原始字节的约 40%，约为全部 268 篇笔记正文（约 4.07 MiB）的 4.2 倍；想瘦身先从 `.obsidian` 插件文件入手，而不是删笔记。
