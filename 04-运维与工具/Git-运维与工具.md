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

- 大文件处理（Git LFS）
- 子模块 vs subtree
