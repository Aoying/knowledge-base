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
