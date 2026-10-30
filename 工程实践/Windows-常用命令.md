---
知识点: Windows 常用命令
层级: 基础
适用版本: 通用
更新日期: 2026-10-30
关联:
  - 工程实践知识点地图
  - Linux与Shell
标签:
  - 运维
  - 速查
---

# Windows 常用命令

> 每晚 20:30 自动补充完善。PowerShell 优先，cmd 备注差异。

## 文件与目录

```powershell
dir / ls                 # 列目录（PS 用 ls/ dir 都是 Get-ChildItem 别名）
cd ~                     # 回用户目录
mkdir newdir             # 新建目录
del file.txt / rm        # 删文件；rmdir /s /q dir  删目录树
copy a b / cp            # 复制；move / mv  移动/重命名
type file.txt / cat      # 看文件内容
findstr "关键字" *.log   # 文件内搜索（PS 可用 Select-String）
tree /f                  # 树形列目录
```

## 网络

```powershell
ipconfig /all            # IP 配置
ping -n 4 host           # ping 4 次
tracert host             # 路由跟踪
netstat -ano | findstr 8080   # 查占用 8080 的进程 PID
nslookup domain          # DNS 查询
```

## 进程与服务

```powershell
tasklist | findstr java        # 进程列表过滤
taskkill /F /PID 1234         # 强杀进程；taskkill /F /IM java.exe
Get-Process java | Stop-Process -Force   # PS 写法
sc query state= all           # 服务列表
net start / net stop 服务名   # 启停服务
```

## 系统信息

```powershell
systeminfo                    # 系统概要（cmd/PS 通用）
winver                        # 弹窗看版本
echo %USERNAME% / $env:USERNAME
set                           # 环境变量列表（cmd）；PS 用 Get-ChildItem Env:
where python                  # 找可执行文件路径
```

## 包管理 winget

```powershell
winget search ollama
winget install Ollama.Ollama
winget upgrade --all          # 全部升级
winget list                   # 已安装列表
```

## 实用组合

```powershell
# 查端口占用并杀掉
$pid=(Get-NetTCPConnection -LocalPort 8080).OwningProcess; Stop-Process -Id $pid -Force
# 批量重命名
Dir *.txt | Rename-Item -NewName { $_.Name -replace '.txt','.md' }
# 看大文件尾部（日志）
Get-Content app.log -Tail 50 -Wait
```

## 常见坑

1. **中文乱码**：cmd 默认 GBK。`chcp 65001` 切 UTF-8；PowerShell 7 默认 UTF-8 无此问题。
2. **PowerShell 执行策略**：脚本跑不起来先 `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`。
3. **路径空格**：`"C:\Program Files\xxx"` 加引号；`&` 调用带空格路径的可执行文件。
4. **换行符 CRLF**：跨平台脚本注意 `\r`，Git 仓库设 `core.autocrlf=true`（Windows 推荐）。
5. **管理员权限**：改 hosts、看系统进程等需要管理员终端（右键以管理员身份运行）。

## 疑难杂症

### 1. WSL2 里连不上 Windows 上的服务
- **现象**：Windows 上起了个服务（localhost:8080），WSL2 里 curl 不通。
- **原因**：WSL2 是独立虚拟网卡，和 Windows 不在同一个 localhost。
- **解决**：WSL2 里用 `$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')` 拿 Windows 宿主机 IP；或在 Windows 防火墙给 WSL 网段放行。

### 2. winget 安装卡住或找不到包
- **现象**：`winget install` 一直转圈，或提示找不到包。
- **原因**：默认源在国外，网络差；或包名不对。
- **解决**：先 `winget search` 确认包名/Id；换源或挂代理；急用直接去 GitHub Releases 下安装包。

## PowerShell 进阶

```powershell
# 找最吃内存的 5 个进程
Get-Process | Sort-Object WS -Descending | Select-Object -First 5 Name, WS

# 递归找大文件（>100MB）
Get-ChildItem -Recurse -File | Where-Object Length -gt 100MB | Sort-Object Length -Descending

# 并行执行（PS7+，多核跑批处理）
1..10 | ForEach-Object -Parallel { Invoke-RestMethod "https://api/api/$_" } -ThrottleLimit 5

# 查端口占用并杀掉
Get-NetTCPConnection -LocalPort 8080 | Select-Object OwningProcess
Stop-Process -Id <pid> -Force

# 环境变量（当前会话 / 永久）
$env:PATH += ";D:\tools"
[Environment]::SetEnvironmentVariable("FOO","bar","User")
```

- `Where-Object`、`Sort-Object`、`Select-Object` 是三件套，管道思维和 Linux 的 `grep|sort|head` 对应。
- PS5.1 和 PS7 语法有差（`-Parallel` 只有 7+ 有），写脚本前先 `$PSVersionTable`。

## 定时任务 schtasks

```cmd
:: 每天 7:30 跑脚本
schtasks /create /tn "MorningReport" /tr "D:\scripts\report.bat" /sc daily /st 07:30
:: 开机自启
schtasks /create /tn "OllamaServer" /tr "ollama serve" /sc onstart /ru SYSTEM
:: 查 / 删 / 立即跑一次
schtasks /query /tn "MorningReport"
schtasks /delete /tn "MorningReport" /f
schtasks /run /tn "MorningReport"
```

- 图形界面是 `taskschd.msc`，命令行适合写进部署脚本。
- 批处理里注意：`%date%` 格式随系统区域变，写日志文件名用 PowerShell 的 `Get-Date -Format yyyyMMdd` 更稳。

## WSL2 进阶

```bash
# systemd（新版 WSL 默认开，可跑 docker）
sudo systemctl status docker
# 和 Windows 互访文件
/mnt/c/Users/ao/Documents        # WSL 看 Windows
\\wsl$\Ubuntu\home\ao          # Windows 看 WSL
# 内存限制（%USERPROFILE%\.wslconfig）
[wsl2]
memory=16GB
processors=8
```

- `.wslconfig` 改完 `wsl --shutdown` 重启生效。
- 代码放 WSL 文件系统里（`~/code`），别放 `/mnt/c`（跨文件系统慢 10 倍，见 Docker 疑难杂症）。

## 基础知识补强

### 核心概念与术语

- 先分清三个东西：终端（窗口程序，如 Windows Terminal）、Shell（解释命令的程序，如 PowerShell、cmd）、命令（真正干活的程序或内置命令）。
- cmd 是老式命令提示符，语法来自 DOS 年代；PowerShell 是现代 Shell，处理的是 .NET 对象而不是纯文本。
- 对象管道是 PowerShell 的核心模型：命令输出的对象带属性，后一个命令直接按属性名取值，不用切字符串。
- 命令的两种来源：内部命令（Shell 自带，如 `cd`、`dir`）和外部程序（如 `git.exe`、`ping.exe`），后者在 PATH 里找。
- PATH：系统找可执行文件的目录清单，一个目录找不到就试下一个，全找不到才报「不是内部或外部命令」。
- 环境变量：Shell 和子进程继承的键值对配置，如 `PATH`、`TEMP`、`USERNAME`；分用户级和系统级两层，系统级要管理员才能改。
- 当前目录：相对路径的参照点，提示符里显示的就是它；每个盘符在 cmd 里还有自己记的当前目录。
- 相对路径与绝对路径：`.\` 是当前目录，`..` 是上级目录，`\` 开头是当前盘根目录，`C:\` 是写死盘符的绝对路径。
- 重定向：把命令的输出送到文件或别处；错误输出是单独的一条通道（stderr），可以单独重定向。
- 退出码（exit code）：命令结束时给出的数字，0 表示成功，非 0 表示失败，脚本靠它判断成败。
- 批处理（.bat/.cmd）是给 cmd 的脚本，PowerShell 脚本是 .ps1，两者语法不通用，别混写。
- 管理员与普通权限是两种令牌：同一条命令在管理员终端能过、普通终端报拒绝访问，通常是权限问题而不是命令错误。

### 基础语法与用法

先认识提示符和最基础的日常操作：

```powershell
# 看自己在哪、目录里有什么
pwd                       # 显示当前目录（Get-Location）
Get-ChildItem             # 列目录，dir / ls 都是它的别名
Get-ChildItem -Force      # 连隐藏文件一起列
cd D:\work                # 切目录；PowerShell 跨盘不用加参数
cd ..                     # 回上级；cd ~ 回用户目录

# 看文件内容与清屏退出
Get-Content a.txt         # 看全文，cat / type 是别名或 cmd 等价物
Get-Content a.txt -TotalCount 20   # 只看前 20 行
cls                       # 清屏
exit                      # 关闭当前 Shell
```

cmd 与 PowerShell 的路径差异（最常踩）：

```cmd
:: cmd 写法
cd /d D:\work             :: cmd 切盘必须加 /d，否则只切了目录没换盘
D:                        :: 单独一行盘符加冒号 = 换盘
echo %PATH%               :: cmd 用 %变量名% 读环境变量
set FOO=bar               :: cmd 设置变量（仅当前窗口有效）
```

```powershell
# PowerShell 写法
$env:PATH                 # PS 读环境变量用 $env: 前缀
$env:FOO = "bar"          # PS 设置变量（仅当前会话有效）
Get-Command git           # 看命令来自哪个文件（等价 where git）
Get-Help Get-Process      # 查命令自带帮助，-Examples 只看例子
Get-History               # 本会话执行过的命令
```

重定向与管道（两种 Shell 通用部分）：

```powershell
dir > list.txt            # 标准输出写进文件（覆盖）
dir >> list.txt           # 追加到文件末尾
ping host > out.txt 2>&1  # cmd 写法：错误输出也并进同一个文件
Get-Process | Where-Object CPU -gt 100 | Select-Object Name, CPU
Get-Process | Measure-Object   # 统计管道里对象的个数
notepad (Get-Command git).Source   # 把上条命令的结果当参数用
```

通配符与引号规则：

```powershell
Get-ChildItem *.log             # * 任意多个字符
Get-ChildItem app?.log          # ? 任意单个字符
Copy-Item "C:\Program Files\a.txt" .   # 路径带空格必须加引号
.\run.ps1                       # 当前目录的脚本要写 .\ 前缀，PS 不自动找当前目录
& "C:\Program Files\app\app.exe"   # & 是调用运算符，执行引号里的程序路径
```

退出码的读法：

```powershell
git status
$LASTEXITCODE             # PS 里外部程序的退出码；成功为 0
$?                        # 上一条命令是否成功（True/False）
```

```cmd
git status
echo %ERRORLEVEL%         :: cmd 里的等价物
```

查信息的基础命令补齐：

```powershell
Get-Service | Where-Object Status -eq Running   # 正在跑的服务
Get-ComputerInfo OsName, OsVersion              # 系统与版本信息
Test-Path D:\work\a.txt                          # 路径是否存在，返回 True/False
Resolve-DnsName example.com                      # DNS 解析（nslookup 的 PS 版）
Test-NetConnection host -Port 3306               # 测某端口通不通（替代 telnet 探活）
```

### 关键机制速览

- PowerShell 管道传对象：`Get-Process` 输出的每一项都有 Name、Id、CPU 等属性，所以能直接 `Sort-Object CPU`；Linux 管道传的是文本，要靠切分字段，这是两者最大的思维差异。
- 别名是双刃剑：`ls`、`cp`、`mv`、`cat` 在 PowerShell 里都是长命令的别名，交互时好用，写脚本时要用全名，否则换台机器别名对不上。
- 变量作用域：`$env:` 改的是进程环境变量，关掉窗口就没了；想永久生效要写用户/系统环境变量（见原文永久设置写法），写完必须新开终端才读得到。
- 执行策略只管 .ps1 脚本文件，不管交互命令；它防的是误双击运行脚本，不是安全边界。
- cmd 的变量展开在整行解析时就完成，所以在 `if` 和括号代码块里改变量再读取会拿到旧值，这是 cmd 脚本最有名的坑。
- 当前目录不进搜索范围：PowerShell 出于安全不在当前目录找程序，必须写 `.\xxx`；cmd 则会找当前目录，两边行为相反。
- 长路径与编码是 Windows 两大历史包袱：路径超过 260 字符、文件名带中文或空格，是各种工具报错的高频源头。

### 新手易错点

- 在 PowerShell 里写 `%PATH%` 读变量：那是 cmd 语法，PS 里要用 `$env:PATH`，写错只会得到一串原样文本。
- 删文件前不确认路径：`Remove-Item -Recurse` 和 cmd 的 `rd /s` 都是不可恢复的递归删除（风险提示：先去掉 -Recurse 或先用 -WhatIf 演练一遍，确认目标目录无误再执行，回收站不会有）。
- 用 `setx` 拼 PATH：`setx` 会截断过长的值且不展开已有变量（风险提示：改 PATH 先备份原值，用图形界面或 .NET 方法按条目追加，别整体覆盖）。
- 把 `.bat` 和 `.ps1` 混着写：`$` 开头的变量、管道对象语法在 cmd 里全不认，反过来 `%VAR%` 在 PS 里也不认。
- 脚本双击闪退就以为没报错：双击运行出错即关窗口，先在终端里手动跑一遍看完整报错和退出码。
- 管理员终端里映射的网络盘符在普通终端看不到：盘符映射按权限令牌隔离，同一路径换终端要重新映射。
- 复制命令时带了中文标点：中文逗号、中文引号粘进终端多半直接语法错误，粘贴后先检查引号和括号是半角。
- `cd` 切了目录却还在原盘：在 cmd 里忘了 `/d` 是最常见的一个，切完用 `cd` 单独回车看当前路径确认。

## 面试与自测

1. 在 PowerShell 里读环境变量为什么不能写 `%PATH%`？cmd 和 PowerShell 设置变量的作用范围各是什么？
2. 在 cmd 里执行 `cd D:\work` 后却还在原盘，为什么？怎么确认切盘成功？
3. WSL2 里连不上 Windows 上起的 localhost:8080 服务，原因是什么？有哪两种解法？

### 参考答案

1. `%PATH%` 是 cmd 语法，PowerShell 里读环境变量要用 `$env:PATH`，写错只会得到一串原样文本。作用范围上，cmd 的 `set FOO=bar` 仅当前窗口有效，PowerShell 的 `$env:FOO = "bar"` 仅当前会话有效；想永久生效要用 `[Environment]::SetEnvironmentVariable("FOO","bar","User")`，写完必须新开终端才读得到。
2. 因为 cmd 切盘必须加 `/d`，不加只切了目录没换盘；单独一行盘符加冒号（如 `D:`）才是换盘，而 PowerShell 跨盘不用加参数。切完用 `cd` 单独回车看当前路径确认，这是最常见的一个坑。
3. 因为 WSL2 是独立虚拟网卡，和 Windows 不在同一个 localhost 上。两种解法：WSL2 里用 `$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')` 拿 Windows 宿主机 IP 来访问；或在 Windows 防火墙给 WSL 网段放行。
