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
