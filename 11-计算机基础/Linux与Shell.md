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
- Windows 对照见 [04-运维与工具/Windows-常用命令.md](04-运维与工具/Windows-常用命令.md)。
