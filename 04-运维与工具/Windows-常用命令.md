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
