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

## Gitee / GitHub 对应

- 两个平台的 git 协议操作完全一样，区别只在远端地址和网页功能。
- 你的 ob_note（Gitee）和 knowledge-base（GitHub）都是私有仓，远端地址别贴到公开地方。

## 待补充


- 子模块 vs subtree

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
