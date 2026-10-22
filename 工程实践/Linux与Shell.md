---
知识点: Linux与Shell
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - 工程实践知识点地图
  - 计算机基础知识点地图
  - Git-运维与工具
标签:
  - 运维
  - 计算机基础
  - 实践
---

# Linux 与 Shell

> 服务端 99% 是 Linux。这篇是"在 Linux 上活下去"的最小集合。

## 文件权限

```bash
-rwxr-xr--  # r=4 w=2 x=1，755 = rwxr-xr-x
chmod 755 app.sh
chown app:app /data/app
```

- `chmod +x` 忘加是新手第一坑。
- 私钥 `~/.ssh/id_rsa` 必须是 600，不然 ssh 拒绝用。

## 文本三剑客

```bash
grep -rn "NullPointer" /data/logs --include="*.log" | head -20
# 统计 500 的 URL Top10
awk '$9==500 {print $7}' access.log | sort | uniq -c | sort -rn | head
sed -i 's/127.0.0.1/10.0.0.5/g' config.properties
find /data -name "*.log" -mtime +7 -delete   # 删 7 天前日志
```

## 管道与重定向

- `|` 传 stdout，`2>&1` 把 stderr 并过去，`tee` 既显示又存文件。
- `xargs`：把管道输出变参数，`find ... | xargs grep xxx`。

## 进程管理

```bash
ps aux --sort=-%cpu | head      # 谁在吃 CPU
kill -15 <pid>                 # 先礼貌，再 kill -9
jobs / fg / bg / Ctrl+Z        # 前后台
nohup ./app.sh &               # 退出终端不死
```

## systemd

```bash
systemctl status/start/stop/restart app
journalctl -u app -f           # 看日志
systemctl enable app           # 开机自启
```

- 写 service 文件比 nohup 靠谱：崩了自动拉起（`Restart=always`）。

## 环境变量与 PATH

```bash
export JAVA_HOME=/usr/lib/jvm/java-17
echo $PATH
```

- 临时 export 只对当前 shell 生效；永久写 `~/.bashrc` 或 `/etc/profile.d/`。

## 磁盘与内存

```bash
df -h        # 磁盘还剩多少
du -sh *     # 谁占的地方大
free -h      # 内存
```

## SSH

```bash
ssh -i key.pem user@host
scp app.jar user@host:/data/
ssh -L 3306:127.0.0.1:3306 user@host  # 本地端口转发，连内网数据库
```

- `-L` 本地转发是连内网 RDS/Redis 最常用的姿势，不用开公网端口。
- Windows 对照见 [工程实践/Windows-常用命令.md](工程实践/Windows-常用命令.md)。


## systemd service 示例（Java 后端最常用）

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My SpringBoot App
After=network.target mysql.service

[Service]
User=app
WorkingDirectory=/data/myapp
ExecStart=/usr/bin/java -Xms1g -Xmx2g -jar /data/myapp/app.jar --spring.profiles.active=prod
Restart=always
RestartSec=5
# JVM 日志 + 应用日志统一走 journald，也可重定向到文件
StandardOutput=append:/data/logs/myapp.log
StandardError=append:/data/logs/myapp.log

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now myapp
journalctl -u myapp -f --since "5 min ago"
```

- `Restart=always` 是生产底线，OOM 被杀、JVM 崩溃都能自动拉起。
- 别在 ExecStart 里写 `cd` 或管道，systemd 不认 shell 语法；要复杂逻辑包一层脚本。

## crontab 定时任务

```bash
crontab -e
# 每天 3 点清 7 天前日志；每周一 4 点重启一次防内存泄漏（无奈但实用）
0 3 * * * find /data/logs -name "*.log" -mtime +7 -delete
0 4 * * 1 systemctl restart myapp
```

- 五段：分 时 日 月 周；`*` 是每。`crontab -l` 看，`*/5 * * * *` = 每 5 分钟。
- cron 里 PATH 很干净，写脚本用绝对路径，不然 `java` 找不到。

## 性能排查三件套

```bash
top              # 按 M 看内存、按 P 看 CPU；load average 三个数看 1/5/15 分钟负载
vmstat 1 5       # r(运行队列)>核数说明 CPU 排队；si/so 有值说明在 swap，内存不够了
iostat -x 1 3    # %util 接近 100 说明磁盘是瓶颈
ss -tlnp         # 谁在监听哪个端口（比 netstat 快）
lsof -i :8080    # 哪个进程占了 8080
```

- Java 排查顺序：`top` 找 PID → `top -Hp <pid>` 看线程 → `jstack <pid>` 抓栈 → 死锁/忙循环一目了然。
- `free -h` 里 available 才是真实可用内存，别看 free 那一列。

## 防火墙

```bash
firewall-cmd --add-port=8080/tcp --permanent && firewall-cmd --reload
# 或 ufw（Ubuntu）
ufw allow 8080/tcp && ufw enable
```

- 云服务器注意：安全组 + 系统防火墙是两层，都要放行，排查连不上时两边都看。

## 常用组合拳

```bash
tar -czvf app-20261019.tar.gz /data/myapp          # 打包
rsync -avz --progress /data/myapp user@host:/data/  # 增量同步，比 scp 快
jq '.data.list[] | .name' resp.json                 # 解析 JSON 接口返回
du -sh /* 2>/dev/null | sort -rh | head             # 磁盘满了先看谁最大
df -i                                              # inode 用完也会写不进文件（小文件太多时）
```

- `du -sh` 找大目录三连：先看根、再看嫌疑目录、最后 `find -size +500M`。
- `tail -f` 看日志卡死时加 `-n 200` 先看尾部，或者 `less +F`。

## 基础知识补强

### 核心概念与术语

- **Shell 是什么**：命令解释器，你敲的命令由它解析执行。服务器上默认多为 bash；脚本第一行写明用哪个解释器，系统才知道怎么跑它。
- **目录结构心智模型**（记高频的几个就够）：
  - `/etc`：系统和软件的配置文件，找配置先来这。
  - `/var/log`：日志集中地；`/var` 本义是经常变化的数据。
  - `/home/用户名`：普通用户的家目录；root 的家目录是 `/root`。
  - `/usr`：系统软件和命令的主体（不是 user 的缩写目录）；`/usr/local` 是手动装的东西。
  - `/opt`：第三方独立软件包常放这；`/tmp` 是临时文件，重启可能被清。
  - `/proc`：虚拟文件系统，里面的文件是内核和进程的实时状态，不是真实磁盘文件。
- **绝对路径与相对路径**：以 `/` 开头的是绝对路径，从根目录算起，写脚本和定时任务必须用它；不以 `/` 开头的是相对路径，从当前目录算起。`.` 表示当前目录，`..` 表示上一级目录，`~` 表示当前用户的家目录。
- **隐藏文件**：文件名以 `.` 开头的文件默认不显示，如 `.bashrc`，多为配置文件。`ls` 不带参数看不到它们不是文件不存在。
- **一切皆文件**：磁盘、设备、进程信息在 Linux 里都以文件形式呈现，所以同一套查看和权限工具能管所有东西。这是理解 Linux 的总纲。
- **标准输入输出**：每个命令默认有三条通道——标准输入（键盘，编号 0）、标准输出（屏幕，编号 1）、标准错误（屏幕，编号 2）。重定向和管道操作的都是这三条通道。
- **退出码**：命令执行完返回一个数字，0 表示成功，非 0 表示出错（具体数值看命令约定）。脚本判断上一步成没成，靠的就是它，变量 `$?` 取最近一个命令的退出码。
- **root 与 sudo**：root 是权限不受限的超级用户。日常用普通用户，需要特权时命令前加 `sudo` 临时提权，出错的影响面小得多；直接长期用 root 登录是高危习惯。
- **发行版与包管理器**：Linux 是内核，各发行版（Ubuntu、Debian、CentOS、Rocky 等）在它之上打包了不同的工具集。装软件用发行版自带的包管理器：Debian 系是 `apt`，Red Hat 系是 `yum` 或 `dnf`。命令不通用，先确认系统是哪一系再查命令。
- **当前目录的重要性**：绝大多数命令的作用对象由当前目录决定。执行删除、解压、改配置前，先 `pwd` 加 `ls` 确认自己站在哪，是成本最低的防呆。

### 基础语法与用法

目录与文件的基本功（先把这组练成本能）：

```bash
pwd                  # 我现在在哪个目录
cd /data/app         # 进绝对路径；cd .. 回上一级；cd ~ 回家目录；cd - 回上次所在目录
ls -la               # 列表含隐藏文件，并显示权限、属主、大小、时间
ls -lh               # 尺寸换成人能读的单位；按时间倒序加 -t
mkdir -p a/b/c       # -p 连着建多级目录，已存在也不报错
touch a.txt          # 建一个空文件（或更新文件的修改时间）
cp -r src dst        # 拷目录必须带 -r 表示递归
mv old new           # 移动或改名，同一目录下用它就是改名
cat small.txt        # 一次性看完小文件全文
less big.log         # 分页看大文件，空格翻页，q 退出，/关键词 搜索
head -n 20 f.log     # 看前 20 行；tail -n 20 是后 20 行
```

删除与脚本的保命写法：

```bash
rm file.txt          # 删文件；rm -r dir 删目录及其内容
# 警告：rm -rf 是最危险的组合，路径写错（如变量为空变成删根目录）后果不可恢复
# 删之前先用 ls 把同一路径列一遍确认，再把 ls 换成 rm 执行
rmdir emptydir       # 只删空目录，非空会报错，日常比 rm -r 更安全
```

```bash
#!/bin/bash
# 脚本第一行叫 shebang，指定解释器，固定写在第一行第一列
set -e               # 任一命令失败立即退出，避免错误滚雪球（简单脚本推荐）
name="world"         # 定义变量：等号两边不能有空格
echo "hello $name"   # 双引号里变量会被替换；单引号里原样输出
echo 'hello $name'   # 输出 hello $name，不替换
./app.sh             # 执行脚本要带路径，或先 chmod +x 后也要写 ./
```

先认识这台机器（接手陌生服务器的第一组命令）：

```bash
whoami               # 当前登录用户是谁
hostname             # 这台机器叫什么
uname -a             # 内核和系统版本信息
cat /etc/os-release  # 发行版名称与版本，确认是哪一系
date                 # 系统时间；时间不对先怀疑时区或未同步
which java           # 某个命令实际执行的是哪个路径下的文件
```

命令连接与通配：

```bash
cmd1 && cmd2    # cmd1 成功才跑 cmd2
cmd1 || cmd2    # cmd1 失败才跑 cmd2
cmd1 ; cmd2     # 不管成败依次都跑
ls *.log        # * 匹配任意多个字符；? 匹配单个字符
ls ?.log
man ls          # 看命令的官方手册；大多数命令 --help 是速览版
history         # 历史命令；Ctrl+R 搜索用过的命令，比重新敲快
```

### 关键机制速览

- **重定向的本质**：`>` 把标准输出改道到文件（覆盖写），`>>` 是追加写，`<` 是把文件当输入。`2>` 单独改道错误输出；`2>&1` 表示把错误输出并到当前标准输出所去的地方，顺序不能颠倒理解：它要在 `>` 之后写才有效。
- **管道的本质**：`|` 把左命令的标准输出接到右命令的标准输入，错误输出不走管道。所以命令报错信息穿过管道还在屏幕上，不是管道坏了。
- **权限三组人**：一串权限按「属主、属组、其他人」三组读，每组各有读写执行。改权限先想清改的是哪组人，滥用 777 是把门全敞开，出事只是时间问题。
- **软链接与硬链接**：软链接（`ln -s 源 链接名`）像快捷方式，存的是路径，源删了它就失效，日常用它切版本目录；硬链接是同一文件多一个名字，不占新空间，不能跨分区，一般不用。
- **变量作用域直觉**：当前 shell 里赋值只在当前 shell 有效，开新终端就没了；`export` 后子进程（你运行的程序）才能继承。程序读不到你定义的变量，先查有没有 export。
- **进程与信号直觉**：进程是正在运行的程序实例，有唯一 PID。`kill` 本质是发信号不是杀人：默认 15 号是请求退出（给进程收尾机会），9 号是强制终结（进程无法拦截，未保存的数据会丢）。

### 新手易错点

- **在 root 下随手操作**：root 下没有权限拦截，路径敲错一格就是全盘事故。日常普通用户加 sudo，并对删除、覆盖类操作保持确认习惯。
- **复制粘贴网上的命令直接回车**：命令里带删除、格式化、往根目录写的内容，粘错或变量未替换会直接执行。长命令先粘到编辑器看一遍再运行，尤其是带 `rm`、`dd`、`mkfs` 字样的。
- **空格导致路径解析错误**：目录名带空格时 shell 会当成两个参数，要加引号或转义。养成习惯：自己建的目录文件名一律不用空格。
- **`cat` 大文件刷屏**：几百兆日志直接 cat 既慢又找不到重点。看日志用 `less` 分页、`tail` 看尾部、`grep` 先过滤，顺序别反。
- **脚本在 Windows 编辑后到 Linux 跑不了**：报莫名其妙的 `bad interpreter` 或每行结尾报错，多半是换行符变成了 CRLF。保存为 LF，或用 `dos2unix` 转一次。
- **以为改了 `.bashrc` 立即生效**：它只在新开 shell 时加载；当前终端要手动 `source ~/.bashrc` 才刷新。改完没反应先想到这一条。
- **相对路径在定时任务和脚本里失效**：手动跑时当前目录是对的，换个执行环境当前目录就变了。脚本里涉及文件一律写绝对路径，或在脚本开头先 `cd` 到脚本所在目录。
