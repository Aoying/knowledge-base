---
知识点: Linux与Shell
层级: 基础
适用版本: 通用
更新日期: 2026-11-10
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

## 复杂日志分析案例

Nginx access 日志默认 combined 格式没有耗时字段，先确认 log_format 里加了 `$request_time`（放末尾），否则排不出慢请求：

```bash
# 状态码分布
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# 慢请求 TopN（$NF 是末尾的 request_time，$7 是请求路径）
awk '{print $NF, $7}' access.log | sort -rn | head -20
# 按小时聚合（$4 是 [19/Oct/2026:14:23:01，截出日:时）
awk '{print substr($4,2,14)}' access.log | sort | uniq -c
```

- 字段位置随 log_format 变，命令里的 `$9`、`$NF` 先拿一条日志数一遍再跑，别照抄。
- 5xx 扎堆先看是某一个接口还是全站：`$7` 分组后再看，别只看总数。

Java 应用日志是多行堆栈，`grep` 会把一次异常拆成多条。先把续行拼回一条记录再统计：

```bash
# 行首是日期的行开新记录，其余行拼到上一条；再抽异常类型计数
awk '/^[0-9]{4}-[0-9]{2}-[0-9]{2}/{if (buf) print buf; buf=$0; next} {buf=buf " | " $0} END{print buf}' app.log \
  | grep -o '[A-Za-z0-9_.]*Exception' | sort | uniq -c | sort -rn
```

- 行首日期正则要按自家日志格式改（logback 默认是 `yyyy-MM-dd HH:mm:ss`，上面按这个写的）。
- 只看异常类型会漏根因：定位到具体某条后，再用时间戳把前后上下文捞出来。

JSON 日志（一行一条，别用多行那套）：

```bash
jq -r '.level' app.json | sort | uniq -c | sort -rn   # 级别分布
jq -r 'select(.duration_ms > 500) | [.timestamp, .msg, .duration_ms] | @tsv' app.json
```

- `jq` 报错先看是不是某行 JSON 被截断：`jq empty app.json` 能定位到坏行。

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

## 排查实战

### Java 进程 CPU 飙高：从进程到代码行

```bash
top -Hp "$PID"            # PID 换成实际进程号，锁定吃 CPU 的线程号（十进制）
printf '%x\n' 12345         # 转 16 进制：3039
jstack "$PID" | grep -A 30 'nid=0x3039'   # 该线程在干什么
```

- 栈停在业务循环里：死循环或正则回溯，看循环条件和量词嵌套。
- 栈多是 GC 线程：`jstat -gcutil <pid> 1000 5` 看 FGC 是不是在涨，堆不够先调大再查代码。
- 栈大量 `BLOCKED`：锁竞争，顺着锁对象找持锁线程。
- `jstack` 要用进程属主用户跑，JDK 的 bin 不在 PATH 里先补路径。

### 磁盘打满：四步定位

```bash
df -h                    # 确认哪个挂载点满了
df -i                    # inode 用完也会写不进文件（小文件太多时）
du -sh /* 2>/dev/null | sort -rh | head   # 从根往下钻，别一上来就 du 全盘
lsof | grep deleted     # 已删但还被进程占着的文件
```

- `du` 算出来远小于 `df` 显示的已用空间，就是“已删未释放”的信号：删文件不回空间。
- 根治办法是重启占着文件的进程；应急可截断 fd（只对普通文件）：`: > /proc/<pid>/fd/<fd号>`，生产上重启更稳。

## 防火墙

```bash
firewall-cmd --add-port=8080/tcp --permanent && firewall-cmd --reload
# 或 ufw（Ubuntu）
ufw allow 8080/tcp && ufw enable
```

- 云服务器注意：安全组 + 系统防火墙是两层，都要放行，排查连不上时两边都看。

## 安全实践

```bash
# /etc/ssh/sshd_config：密钥登录 + 禁用密码登录
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin prohibit-password
systemctl reload sshd
```

- **先开新会话验证密钥能登录，再关密码登录**，顺序反了就把自己锁在门外。
- sshd 加固：`AllowUsers` 白名单只放运维账号；改默认端口只能挡扫描，别当安全措施；登录失败看 `/var/log/auth.log`（Debian 系）或 `/var/log/secure`（RHEL 系）。

sudo 最小权限：别给应用账号 `ALL NOPASSWD`，只放它必须跑的那条命令：

```text
# /etc/sudoers.d/app，用 visudo 改，语法错了 sudo 直接罢工
app ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp
```

防火墙默认拒绝，再按需放行：

```bash
ufw default deny incoming
ufw allow 22/tcp && ufw allow 8080/tcp
ufw enable   # 先放行 SSH 再 enable，不然当前连接就断了
```

- fail2ban 一句话：自动封禁暴力破解的 IP，`fail2ban-client status sshd` 看封禁状态。

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

## 小技巧与冷知识

【小技巧】上条命令跑完才发现忘了 sudo，别重新敲一遍：直接输入 `sudo !!`，`!!` 会被 shell 展开成上一条命令，等于原样加 sudo 再跑一次。同理 `!grep` 是重跑最近一条以 grep 开头的命令，`history` 里翻到编号后 `!123` 能直接重跑第 123 条；历史命令不只是用来看的。

【小技巧】两个目录来回切别反复敲绝对路径，用目录栈：`pushd /data/logs` 进目录并把当前位置压栈，`popd` 回到上一个目录，`dirs` 看栈里还压着哪些目录。原文只讲了 `cd -` 只能回上一次，pushd/popd 能在三四个目录之间轮转，排查时在日志目录、配置目录、代码目录之间跳最省事。

【小技巧】文件名带空格时，`find ... | xargs` 会把一个文件拆成两个参数：先用 `find /data -name "*.log" -print0 | xargs -0 rm`，`-print0` 用空字符分隔文件名、`-0` 让 xargs 按空字符切分，空格就不再是分隔符。原文提醒“自己建的目录文件名一律不用空格”，但接手别人的机器时空格文件已经在那了，这组参数就是兜底。

【小技巧】脚本跑偏了先别加 echo 满屏打点，用 `set -x`：在脚本开头或出错位置前加一行 `set -x`，之后每条命令执行前都会把原命令打印出来再跑，变量展开后的实际值也一并可见。原文推荐的 `set -e` 是失败即停，`set -x` 是边跑边报，两个配合用——先看清它到底跑了什么，再谈为什么失败。

【小技巧】长时间任务别只靠 nohup，用 tmux：`tmux new -s deploy` 开一个会话跑部署，断开 SSH 后任务还在会话里跑，回来 `tmux attach -t deploy` 就接回原来的屏幕输出。nohup 只能保住进程不死，输出还得事后翻文件；tmux 是把整个终端会话留在了服务器上，交互式任务（比如等输入密码）也能断线重连。

【小技巧】密钥登录别手动把公钥粘进 authorized_keys，用 `ssh-copy-id`：`ssh-copy-id -i ~/.ssh/id_rsa.pub user@host` 会自动把公钥追加到对方 `~/.ssh/authorized_keys`，权限也一并处理好。手动粘贴容易多换行、少字符或权限不对，配完登录失败先查的还是权限；一个命令交出去，出错面少一半。

【冷知识】`rm -rf /` 在现代系统上默认会被拒绝：GNU coreutils 从 2006 年起把 `--preserve-root` 作为默认行为，`rm -rf /` 会直接报错退出，必须显式加 `--no-preserve-root` 才删得动。原文警告“rm -rf 路径写错后果不可恢复”依然成立——但删根目录这一下本身有最后一道保险，真正危险的是变量为空时路径变成 `/data` 之类、删了半个业务目录，那层保险拦不住。

【冷知识】`/dev/null` 是个“黑洞”设备文件：往里写多少都直接丢弃，读它立刻返回文件结束。`命令 > /dev/null 2>&1` 的意思就是标准输出和错误输出都扔掉、只留退出码给脚本判断。理解了它就明白，重定向的目标不一定是真实文件，`/dev/null`、`/dev/stdout` 这类设备文件本身就是 Linux“一切皆文件”的实例。

【小技巧】服务起不来先别逐个 systemctl status，用 `systemctl --failed`：一条命令列出所有启动失败的单元，哪个 service 挂了、挂在哪一目了然。原文讲了 `systemctl status/start/stop` 单个服务的操作，`--failed` 是先扫全局再下钻的入口，机器刚重启、批量部署后先跑它，比一个一个查快得多。

## 资源用量定量：文件描述符、进程数与磁盘 inode 算一算

> 建议插入位置：《Linux与Shell》「面试与自测」之前。本节只做定量换算，不重复原文关于文件权限、文本三剑客、日志分析、管道与重定向、进程管理、systemd、crontab、性能排查、防火墙与安全实践的定性结论。
>
> 来源与核验日期：本节数字分两类——（1）已验证事实：引自 kernel.org sysctl/fs 文档（Documentation/admin-guide/sysctl/fs.rst）与 mke2fs 手册、`/etc/mke2fs.conf` 默认值；（2）本机实测例：2026-11-10 在一台 8GB 内存虚拟机（nproc = 4，MemTotal 8,124,900 kB）上实测。所有换算均用 Python 当场复算，公式与中间值写在正文里；凡未标注“本机实测”的数字均为按已验证默认值换算的通用例，不是实测值。

### 1. 每个进程的 fd 预算：连接数吃掉多少，剩多少给文件与管道

文件描述符（fd）是进程级资源，受 `ulimit -n`（软/硬限制）约束；系统级上限是 `file-max`，见第 2 节。本机实测（2026-11-10）：`ulimit -n` 软硬均为 4096；`/proc/sys/fs/nr_open = 1,048,576`（即 2^20，是单个进程理论上能打开的 fd 数上限，来源 kernel.org sysctl/fs 文档）。

换算公式：`剩余 fd = ulimit -n − 已占连接数`。

- 旧默认值场景（`ulimit -n = 1024`）：一个开 500 个连接的 Java 进程，`1024 − 500 = 524`，只剩 524 个 fd 给文件、管道与日志，约占一半（524 ÷ 1024 ≈ 51.2%）。
- 本机实测场景（`ulimit -n = 4096`）：同场景 `4096 − 500 = 3596`，剩 3596 个 fd，约占 87.8%（3596 ÷ 4096 ≈ 87.8%）。

预算表思路（按“连接数 + 日志文件 + jar/类文件 + 管道”四项列预算，再留余量）：

| 预算项 | 说明 | 估算方法 |
| --- | --- | --- |
| 网络连接 | TCP 连接、数据库连接池 | 按峰值连接数计，如 500 |
| 日志文件 | 应用日志、gc 日志等打开的文件句柄 | 按同时打开的日志文件数计 |
| jar / 类文件 | JVM 加载的 jar 与类文件句柄 | 启动后用 `ls /proc/<pid>/fd \| wc -l` 实测基线 |
| 管道 | 进程间管道、子进程标准流 | 按管道对数计 |
| 余量 | 突发连接、临时文件、动态加载 | 至少留 10%–20%，1024 上限时建议不少于 100 个 |

实操对照：`ls /proc/<pid>/fd | wc -l` 看当前已用 fd 数，`ulimit -n` 看上限；已用数接近上限时先按上表分项排查，而不是直接调大限制（调大方法与软硬限制区别见第 6 节边界条件）。

### 2. file-max（系统级）vs ulimit（进程级）：谁先耗尽

`file-max` 是全系统所有进程能同时打开的 fd 总数上限，默认值约为内存的 10%（按 KB 计），来源 kernel.org sysctl/fs 文档（Documentation/admin-guide/sysctl/fs.rst）。

换算公式：`file-max ≈ 内存（KB）÷ 10`。

- 8GB 内存通用例：8 GiB = 8 × 1024 × 1024 = 8,388,608 KB，`8,388,608 ÷ 10 = 838,860`（约 838,860）。
- 本机实测例：本机 MemTotal = 8,124,900 kB，`8,124,900 ÷ 10 = 812,490`，与通用例同量级（差异来自本机可用内存略小于标称 8 GiB）。

全系统最多多少个进程能同时开满 fd？换算公式：`进程数 ≈ file-max ÷ 单进程 ulimit -n`。

- `838,860 ÷ 4096 ≈ 204.8`，即约 204 个进程同时开满 4096 个 fd，就会触及系统级 file-max。
- 对照：本机实测 `/proc/sys/kernel/pid_max = 4,194,304`，即系统允许的进程号上限约 419 万，远大于 204。结论：fd 先耗尽，进程数上限不是瓶颈——报“Too many open files”时先查 fd，而不是查进程数。

注意：本机 `/proc/sys/fs/file-max` 读出 `9223372036854775807`，这是容器环境的人为设定，不是常规默认值，不要把它当典型值。正文换算一律用按内存 10% 算出的约 838,860（8GB 机器）。

### 3. inode 耗尽换算：盘没满也会报 ENOSPC

ext4 的 inode 总数在格式化时按 bytes-per-inode 决定，默认 bytes-per-inode = 16,384（来源：`/etc/mke2fs.conf` 默认值与 mke2fs 手册），格式化后不可改；inode 大小默认 256 字节。

换算公式：`inode 总数 = 磁盘容量（字节）÷ 16,384`。

- 1TB 盘：`1024^4 ÷ 16,384 = 1,099,511,627,776 ÷ 16,384 = 67,108,864` 个 inode。
- 100GB 盘：`100 × 1024^3 ÷ 16,384 = 107,374,182,400 ÷ 16,384 = 6,553,600` 个 inode。

小文件多到什么程度会耗尽 inode？换算公式：`耗尽时已用容量 = inode 总数 × 平均文件大小`。

- 平均每文件 4KB 时（1TB 盘）：`67,108,864 × 4096 = 274,877,906,944 字节 = 256 GiB`，约占 1TB 盘的 25%（256 ÷ 1024 = 25%）。也就是说盘只用了约四分之一就可能因 inode 耗尽而报 ENOSPC（No space left on device），而不是盘满。
- 平均每文件 100 字节时（1TB 盘）：`67,108,864 × 100 = 6,710,886,400 字节 = 6.25 GiB`（约 6.71 GB），即只写约 6.25 GiB 的小文件就可能耗尽全部 inode。

对照原文：`df -i` 查 inode 使用率，`df -h` 查容量使用率；两者要一起看——容量有余量但 inode 用满时，同样写不进文件。

### 4. 小文件块浪费：逻辑大小与块占用差多少

ext4 默认块大小 4KB：一个文件无论多小，至少占一个块。换算以 100 万个 100 字节文件为例：

- 逻辑大小：`1,000,000 × 100 = 100,000,000 字节 ÷ 1024^2 ≈ 95.37 MiB`。
- 块占用：`1,000,000 × 4096 = 4,096,000,000 字节 ÷ 1024^3 ≈ 3.81 GiB`。
- 浪费比例：`(4,096,000,000 − 100,000,000) ÷ 4,096,000,000 × 100% ≈ 97.56%`（约 97.6%），即近 98% 的块空间被小文件的块对齐吃掉了。

这与原文“已删未释放”排查呼应但不重复：原文讲的是删除后句柄未释放导致空间不回收；这里讲的是即使文件都在、未删除，小文件本身的块对齐就会让实际占用远大于逻辑大小。排查时 `du`（块占用）与 `ls -l`（逻辑大小）对不上，先看平均文件大小与文件个数，再决定是清小文件还是改存储方式（如打包、归档）。

### 5. 管道容量：生产者写多少会阻塞

管道默认容量 64KB，本机实测（2026-11-10，fcntl 实测）：管道默认容量 65,536 字节，PIPE_BUF = 4096 字节（PIPE_BUF 是保证原子写入的单次上限，与容量是两个概念，不要混淆）。

换算：生产者写 1MB 数据、消费者不读时，写满 64KB 即阻塞——`65,536 字节`写满后，`write` 阻塞等待消费者读走数据，之后才能继续写。

10MB 日志流经管道至少分几次写？换算公式：`次数 = 总字节 ÷ 管道容量 = 10,485,760 ÷ 65,536 = 160`，即至少 160 次写（每次写满 64KB 的理想情况；实际次数只多不少，取决于每次写入大小与消费速度）。

对照原文“管道与重定向”一节：管道是流式通道，不是无限缓冲；大日志流经管道时，生产者与消费者速度不匹配就会互相阻塞，排查时先确认哪一端停了。

### 6. 边界条件

1. `file-max` 是系统启动时按内存算的默认值，容器内读数不作数——本机读出 9223372036854775807 即为例，换算请用“内存（KB）÷ 10”的公式，不要直接引用容器内的读数。
2. inode 总数在格式化时由 bytes-per-inode 决定，格式化后不可改；已用满 inode 的文件系统只能删文件、迁移数据或重建文件系统，无法在线扩 inode。
3. 块大小同样在格式化时定，事后不可改；小文件多的场景在格式化前就要选小块或改存储方案，事后只能靠归档、打包减少文件个数。
4. `ulimit -n` 的软限制可在会话内调大，但不得超过硬限制；硬限制需 root 权限才能调大，普通用户只能在硬限制内调软限制；systemd 管理的服务用 `LimitNOFILE` 配置，不走 shell 的 ulimit。
5. `threads-max` 随内存变化，本机实测 `/proc/sys/kernel/threads-max = 54,269` 只作本机例（8GB 内存虚拟机，2026-11-10），不要当作通用默认值；线程数上限与 fd 上限是两套独立限制，排查时分开查。

## 面试与自测

1. 私钥 `~/.ssh/id_rsa` 的权限为什么必须是 600？权限数字是怎么算的？
2. Java 进程 CPU 飙高时，怎样从进程定位到具体的代码行？看到不同形态的栈分别说明什么？
3. `df -h` 显示磁盘满了，但 `du` 算出来的占用远小于已用空间，这说明什么？根治办法和应急办法各是什么？
4. 一台 8GB 内存的机器上，`file-max` 默认值大约是多少？一个开 500 个连接的 Java 进程在 `ulimit -n = 1024` 时还剩多少 fd？

### 参考答案

1. 权限必须是 600，不然 ssh 会拒绝用这个私钥。数字按 r=4、w=2、x=1 算，600 就是属主可读写、属组和其他人没有任何权限；同理 755 = rwxr-xr-x。另外脚本执行前别忘了 `chmod +x`，这是新手第一坑。
2. 三步：`top -Hp <pid>` 锁定吃 CPU 的线程号（十进制）→ `printf '%x\n'` 转成 16 进制 → `jstack <pid> | grep -A 30 'nid=0x...'` 看该线程在干什么。栈停在业务循环里是死循环或正则回溯；栈多是 GC 线程则用 `jstat -gcutil <pid> 1000 5` 看 FGC 是否在涨；栈大量 BLOCKED 是锁竞争，顺着锁对象找持锁线程。注意 jstack 要用进程属主用户跑。
3. 说明存在“已删未释放”的文件：文件被删了但还被进程占着，空间不会回来，用 `lsof | grep deleted` 能定位。根治办法是重启占着文件的进程；应急可截断 fd（只对普通文件）：`: > /proc/<pid>/fd/<fd号>`，但生产上重启更稳。另外定位磁盘满要先 `df -h` 确认挂载点、`df -i` 查 inode，再从根往下 `du` 钻，别一上来就 du 全盘。
4. `file-max` 默认值约为内存的 10%（按 KB 计），8GB 内存约 838,860；`ulimit -n = 1024` 时 500 个连接占掉一半，剩 524 个 fd 给文件、管道与日志（约 51.2%），`ulimit -n = 4096` 时剩 3596 个。报 "Too many open files" 时先查 fd 预算：系统级 file-max 约 838,860 ÷ 单进程 4096 ≈ 204 个进程同时开满即触顶，fd 先耗尽、进程数上限（pid_max 约 419 万）不是瓶颈。
