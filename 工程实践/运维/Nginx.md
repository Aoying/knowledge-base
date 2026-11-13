---
知识点: Nginx
层级: 基础
适用版本: 通用
更新日期: 2026-11-14
关联:
  - 工程实践知识点地图
  - Docker
标签:
  - 运维
  - 实践
---

# Nginx

> 每晚 20:30 自动补充完善。

## 反向代理最小可用

```nginx
server {
    listen 80;
    server_name app.example.com;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- `proxy_set_header` 三件套不写，后端拿到的都是 127.0.0.1，日志和限流全错。

## 负载均衡

```nginx
upstream backend {
    server 127.0.0.1:8080 weight=3;
    server 127.0.0.1:8081;
    # ip_hash;        # 按 IP 固定后端（会话保持的土办法）
    # least_conn;     # 最少连接
}
server { location / { proxy_pass http://backend; } }
```

- 默认轮询；`weight` 加权；后端挂了自动剔除（`max_fails`/`fail_timeout` 可调）。

## location 匹配顺序（面试常考）

1. `=` 精确匹配
2. `^~` 前缀匹配（不再看正则）
3. `~` / `~*` 正则（区分/不区分大小写）
4. 普通前缀 / 通用 `/`

## 常用片段

```nginx
# 静态资源缓存
location ~* \.(js|css|png|jpg)$ { expires 30d; add_header Cache-Control "public"; }
# gzip
gzip on; gzip_types text/css application/javascript application/json;
# 上传大小（默认 1m，传文件必改）
client_max_body_size 50m;
# WebSocket 升级
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

## 常见坑

1. **改完配置没 reload**：`nginx -t` 先验语法，再 `nginx -s reload`。
2. **proxy_pass 斜杠**：`proxy_pass http://b/` 和 `http://b` 转发后的 URI 拼接规则不一样，复制配置时注意尾巴的 `/`。
3. **413 上传失败**：`client_max_body_size` 默认 1m。
4. **后端拿不到真实 IP**：漏了 `X-Forwarded-For` 那三行。
5. **Windows 下跑**：能用但生产别用，信号和性能都不如 Linux。


## 疑难杂症

### 1. WebSocket 代理连上就断
- **现象**：ws 直连后端正常，走 Nginx 握手成功后很快断开。
- **原因**：没透传 `Upgrade` 和 `Connection` 头，Nginx 把它当普通 HTTP 处理了。
- **解决**：
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

### 2. location 匹配优先级深坑
- **现象**：配了多个 location，请求总走到不对的那个。
- **原因**：优先级是 `=` > `^~` > 正则（按文件顺序）> 前缀最长；很多人以为按书写顺序。
- **解决**：精确匹配用 `=`，前缀想截胡用 `^~`，正则注意先后顺序；`nginx -T` dump 出来逐条对。

### 3. 反向代理下载大文件卡住
- **现象**：下载几百 MB 的文件，进度条走走停停，甚至超时。
- **原因**：`proxy_buffering` 开着，Nginx 先把后端响应全 buffer 到磁盘/内存再转发。
- **解决**：大文件下载的 location 关掉 `proxy_buffering off`，改走 `X-Accel-Redirect` 让 Nginx 直接发文件。

## HTTPS（certbot 自动续期）
```nginx
server {
    listen 443 ssl http2;
    server_name app.example.com;
    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
    # http 跳 https
}
server { listen 80; server_name app.example.com; return 301 https://$host$request_uri; }
```
```bash
certbot --nginx -d app.example.com   # 一条命令签发+配好
certbot renew --dry-run               # 续期演练，cron 里加 renew 即可
```

## 限流防刷
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```
- `rate=10r/s` 每秒 10 个；`burst=20 nodelay` 允许突发 20 个立即处理，超了 503。

## 反向代理缓存（proxy_cache）

```nginx
proxy_cache_path /data/nginx-cache levels=1:2 keys_zone=api:50m max_size=1g inactive=60m;
server {
    location /api/hot/ {
        proxy_cache api;
        proxy_cache_valid 200 5m;      # 成功响应缓存5分钟
        proxy_cache_key "$scheme$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status;  # HIT/MISS 看缓存是否生效
        proxy_pass http://backend;
    }
}
```

- 读多写少、允许分钟级延迟的接口（如排行榜、配置下发）最适合。
- 调试先看 `X-Cache-Status` 是 HIT 还是 MISS，别上来就怀疑人生。
- 带用户态的接口（`Authorization` 头）默认不缓存，想缓存要显式 `proxy_cache_key` 带上区分维度。

## 动静分离 + 防盗链

```nginx
server {
    # 静态走本地，动态走后端
    location ~* \.(jpg|css|js|woff2)$ {
        root /data/static;
        expires 30d;
        add_header Cache-Control "public";
    }
    location / { proxy_pass http://backend; }

    # 图片防盗链
    location ~* \.(png|jpg)$ {
        valid_referers none blocked *.mydomain.com;
        if ($invalid_referer) { return 403; }
    }
}
```

- `~*` 正则不区分大小写，静态资源后缀匹配用它。
- `expires 30d` 让浏览器缓存，静态流量直接砍 90%。

## 日志切割 + 监控

```nginx
log_format main '$remote_addr - $request_time $status "$request"';
access_log /var/log/nginx/access.log main buffer=64k flush=5s;
```
```bash
# logrotate 每天一切（/etc/logrotate.d/nginx）
/var/log/nginx/*.log { daily; rotate 30; compress; missingok; }
```

- `buffer=64k flush=5s`：攒批写，QPS 高了不加这个磁盘 IO 直接起飞。
- `$request_time` 记上，慢请求分析 `awk '$2>1'` 一把梭。
- 监控：`stub_status` 开个 `/nginx_status`，Prometheus 抓 active connections。

## 基础知识补强

### 核心概念与术语

- **Nginx 是什么**：高性能的 Web 服务器软件，同时常被用作反向代理和负载均衡器。一个软件三种常见角色，别混淆：
  - **静态 Web 服务器**：直接把硬盘上的 HTML、图片、前端打包产物返回给浏览器，不经过后端应用。
  - **反向代理**：站在后端应用前面接收请求再转发，客户端只知道 Nginx，不知道后端在哪。
  - **负载均衡器**：把请求按规则分发到多台后端，是反向代理的延伸用法。
- **正向代理 vs 反向代理**：正向代理代理的是客户端（替你上网，服务端不知道你是谁）；反向代理代理的是服务端（替后端接客，客户端不知道后端是谁）。Nginx 做的是反向代理。
- **指令与配置块**：配置由一条条指令组成。简单指令是 `名字 参数;`，以分号结尾；配置块是 `名字 参数 { ... }`，里面嵌套其他指令。少个分号是新手最常见的语法错误。
- **worker 进程**：Nginx 运行时是一个 master 进程加多个 worker 进程。master 负责读配置、管 worker，不处理请求；worker 才是真正接收和处理连接的。改配置 reload 时 master 让新 worker 用新配置、旧 worker 处理完存量连接再退出，所以 reload 不断连接。
- **server 块**：一个虚拟主机配置，对应一个站点（按域名和端口区分）。一台 Nginx 里可以有很多个 server 块。
- **location 块**：server 内部按 URL 路径分流的规则块，决定这类请求是读静态文件、转发后端还是直接返回。
- **root 与 alias**：都是把 URL 路径映射到磁盘目录。`root` 是把完整 URI 拼到 root 后面；`alias` 是把 location 匹配掉的那段前缀替换掉。两者拼接规则不同，混用最容易 404。

### 基础语法与用法

先看配置文件的骨架层级，任何配置都只能写在它该在的层级里：

```nginx
# 主配置 nginx.conf
user  nginx;                    # 全局层：worker 用什么用户跑
worker_processes  auto;         # worker 数量，auto 表示按 CPU 核数
error_log  /var/log/nginx/error.log;
pid        /run/nginx.pid;

events {                        # 事件层：连接处理模型，只能写连接类参数
    worker_connections  1024;   # 单个 worker 最多同时处理多少连接
}

http {                          # HTTP 层：所有 Web 服务配置的根
    include       mime.types;   # 文件扩展名到响应类型的对照表
    default_type  application/octet-stream;
    sendfile      on;           # 静态文件零拷贝发送，保持开

    server {                    # 站点层：一个域名一个 server
        listen       80;
        server_name  example.com;

        location / {            # 路径层：以 / 开头的所有请求
            root   /data/www;   # 去这个目录找文件
            index  index.html;  # 访问目录时默认找这个文件
        }
    }
}
```

最基础的静态站点，只有三行核心配置：

```nginx
server {
    listen 80;
    server_name example.com;
    location / {
        root /data/www;         # 网站文件放这个目录
        index index.html;       # 不写文件名时默认返回它
    }
}
```

日常操作命令（改配置的标准动作是先测再加载）：

```bash
nginx -t          # 只测试配置语法对不对，不动正在跑的服务
nginx -s reload   # 语法没问题后平滑重载，新请求走新配置，不断连接
nginx -s stop     # 快速停止（直接终止进程，可能掐断正在处理的请求）
nginx -s quit     # 优雅停止（处理完已有连接再退出）
nginx -T          # 打印全部生效配置（含被 include 进来的），排查以它为准
systemctl restart nginx   # 完全重启，会断连接；改配置优先用 reload
```

两个高频基础指令：

```nginx
# 访问根路径时直接找后端目录下的首页，找不到再返回 404 页
location / {
    try_files $uri $uri/ =404;
}

# 统一错误页：后端返回 500 系列时展示自己的页面
error_page 500 502 503 504 /50x.html;
```

- `try_files` 按从左到右顺序试：先当文件找，再当目录找，都没有就返回最后一个状态码。前后端分离项目里常写成找不到就回退到 `index.html`，让前端路由接管。
- `return 301 https://example.com$request_uri;` 是最简单的跳转写法，`301` 是永久重定向，`302` 是临时。

### 关键机制速览

- **请求处理流程**：连接进来先按端口和 `Host` 头选中一个 server 块，再在 server 内选出最匹配的 location，最后按该 location 的指令执行：读文件、转发或直接返回。全程没有匹配上时走默认处理。
- **默认 server**：同一个端口下，域名对不上任何 `server_name` 时，请求落到该端口的第一个 server 块（或显式标了 `default_server` 的那个）。用 IP 直接访问看到意外页面，多半是落到了默认 server。
- **变量**：配置里的 `$host`、`$remote_addr`、`$request_uri`、`$scheme` 等是 Nginx 内置变量，运行时替换为当前请求的实际值。写日志格式、转发头、跳转地址都靠它们，不用背全，用时查。
- **include 机制**：主配置常用 `include conf.d/*.conf;` 把每个站点的配置拆成单独文件。主文件找不到你的 server 时，先查 include 的目录和文件名后缀对不对。
- **静态文件为什么快**：`sendfile on` 时文件内容不经过用户态反复拷贝，内核直接把文件发到网卡。静态资源交给 Nginx 而不是后端应用，图的就是这个。
- **upstream 与转发的关系**：`proxy_pass` 写单个地址就是简单转发；写成一组命名的后端集合（upstream）再引用，Nginx 才能在多台之间做分发和故障剔除。

### 新手易错点

- **改完配置直接 restart**：语法有错时 restart 会让服务起不来直接宕机。固定动作：`nginx -t` 通过后再 `nginx -s reload`，顺序不能反。
- **分号和花括号漏写**：配置不是脚本，指令必须以分号结尾，块必须闭合。报错信息会给出文件名和行号，顺着行号往上找通常是上一行漏了分号。
- **root 写在错误层级理解偏差**：location 里的 root 拼接的是完整请求路径。例如 `root /data;` 加请求 `/img/a.png`，找的是 `/data/img/a.png`；想把 `/img/` 这段去掉要改用 alias，别硬试 root。
- **index 不生效**：访问目录返回 403，多半是目录里没有 index 指定的文件，或者目录权限不允许 Nginx 的运行用户读取。先查文件在不在，再查权限链每一层目录有没有执行权限。
- **把后端端口暴露的锅扣给 Nginx**：请求绕过 Nginx 直连后端端口能通，是后端监听和防火墙的问题，Nginx 配置里没有能拦这个的指令。
- **server_name 写了域名但用 IP 测试**：按域名选 server 靠的是请求里的 Host 头，浏览器输 IP 时 Host 就是 IP，自然落到默认 server。用域名测试，或手动带 Host 头验证。
- **日志不看 error_log 只盯 access_log**：access_log 记录请求结果，error_log 才记录为什么出错（文件找不到、后端连不上、配置问题）。排障先看 error_log 的最后几行。

## 原理剖析

本节只把原文已有章节给出的结论拆开讲清机制：不引入新指令，不补新数字，数字一律复用「连接数与缓存定量」节已核验的算例；凡原文没展开的点，文中明确标注，不作定论。

### master/worker 进程模型：为什么 reload 不断连接

本节只综合原文“基础知识补强”与“常用片段”的结论。Nginx 运行时是一个 master 进程加多个 worker 进程：master 负责读配置、管 worker，不处理请求；worker 才是真正接收和处理连接的。改配置 reload 时 master 让新 worker 用新配置、旧 worker 处理完存量连接再退出，所以 reload 不断连接——这就是原文要求“先 `nginx -t` 再 `nginx -s reload`，而不是直接 restart”的原理：restart 是完全重启，会断连接，且语法有错时服务会起不来直接宕机。worker 数量由 `worker_processes auto` 按 CPU 核数决定，4 核就是 4 个 worker；每个 worker 能同时打开的连接数由 `worker_connections` 封顶，最大并发连接数 = worker_processes × worker_connections。worker 数与连接上限是两层不同的天花板：worker 少了，连接上限再高也接不满；连接上限低了，worker 再多也顶在公式上。

### location 匹配顺序：优先级不是书写顺序

本节只综合原文“location 匹配顺序”与“疑难杂症”第 2 条的结论。优先级是 `=` 精确匹配 > `^~` 前缀匹配（不再看正则）> `~` / `~*` 正则（按文件顺序）> 普通前缀 / 通用 `/`。`^~` 的作用是截胡：某个前缀一旦被 `^~` 命中，就不再看正则，直接用它；想让某个前缀截胡正则时用 `^~`。正则之间按文件里的先后顺序裁决，写在前面的先赢——这是“配了多个 location，请求总走到不对的那个”最常见的根因：很多人以为按书写顺序，实际是按这条优先级表。排障时以 `nginx -T` 打印出的全部生效配置为准逐条对，不要只看自己写的那个文件。`root` 与 `alias` 的拼接规则也在这条链上：`root` 是把完整 URI 拼到 root 后面，`alias` 是把 location 匹配掉的那段前缀替换掉，两者混用最容易 404。

### 反向代理的转发链路：请求头不透传，后端看到的全是错的

本节只综合原文“反向代理最小可用”与“常见坑”的结论。`proxy_pass` 只决定请求发给谁，`proxy_set_header` 三件套（`Host $host`、`X-Real-IP $remote_addr`、`X-Forwarded-For $proxy_add_x_forwarded_for`）决定后端看到的请求头是什么：不写这三行，后端拿到的客户端 IP 全是 127.0.0.1，日志和限流全错。`proxy_pass` 尾巴的斜杠决定 URI 怎么拼：带斜杠时 location 匹配到的那段会被替换掉，不带斜杠时完整原始 URI 原样转发（小技巧）。连接数上反向代理要占两份：官方文档明确说 `worker_connections` 计入全部连接，一个反向代理请求通常占 2 个连接——客户端—Nginx 一个，Nginx—后端一个，所以 4 核、每 worker 1024 个连接时静态直出可接约 4096 个客户端，反向代理只剩约 2048 个。`proxy_buffering` 开着时 Nginx 先把后端响应全 buffer 到磁盘/内存再转发，大文件下载会走走停停甚至超时，关掉它或改走 `X-Accel-Redirect` 让 Nginx 直接发文件（疑难杂症第 3 条）。

### 负载均衡：分发规则与故障剔除是一体的

本节只综合原文“负载均衡”一节的结论。默认轮询，`weight` 加权，后端挂了自动剔除（`max_fails`/`fail_timeout` 可调）。`ip_hash` 按 IP 固定后端，是会话保持的土办法；`least_conn` 发给当前连接最少的后端。分发规则决定“这次请求去哪”，故障剔除决定“挂了的后端不再接新请求”，两者缺一不可：剔除靠 `max_fails`/`fail_timeout` 的阈值，阈值配得太松，挂掉的后端还会继续收到请求；分发策略不选，流量分布就完全听天由命。`proxy_pass` 写单个地址就是简单转发，写成一组命名的后端集合（upstream）再引用，Nginx 才能在多台之间做分发和故障剔除（基础知识补强“关键机制速览”）。upstream 里想复用上游连接还要另配三行（`keepalive` + `proxy_http_version 1.1` + `Connection` 头清空），缺一不可（小技巧），原文的 WebSocket 一节用的 1.1 和 Upgrade 头是升级协议，不是连接复用。

### 缓存命中与失效：keys_zone 管 key，max_size 管内容

本节只综合原文“反向代理缓存”与定量节的结论。`proxy_cache_path` 里两个尺寸管的是两样东西：`keys_zone` 是共享内存，只存缓存 key 和元数据，不存响应体；`max_size` 是磁盘上限，存真正的缓存文件，超了由 cache manager 按最近最少使用（LRU）淘汰。官方换算口径：开源版 1MB 的 keys_zone 大约能存 8000 个 key，相当于每个 key 约 131 字节。原文配置 `keys_zone=api:50m max_size=1g`：50m keys_zone 约 40 万个 key；平均 20KB 时 1g 磁盘约放 52428 个文件，只需约 6.6MB 的 keys_zone，配 10m 绰绰有余；平均 2KB 时约 524288 个文件，需要约 65.5MB 的 keys_zone，50m 反而不够——磁盘还没满，key 先被 LRU 挤掉，命中率就上不去。命中率的账按回源量算：回源量 = 总请求量 ×（1 − 命中率），一天 100 万次请求，命中率 90% 时回源 10 万次，99% 时回源 1 万次，相差 10 倍；1000r/s 的接口，90% 命中时回源 100r/s，99% 时只有 10r/s。“命中率从 90% 提到 99%”真正的意思是“回源量砍掉 90%”，而不是“只好了 9%”。带用户态的接口（`Authorization` 头）默认不缓存，想缓存要显式在 `proxy_cache_key` 里带上区分维度；调试先看 `X-Cache-Status` 是 HIT 还是 MISS，别上来就怀疑人生。

## 连接数与缓存定量：worker_connections 与 proxy_cache 算一算
本节只算数，不讲新指令。先记住口径，再套算例。
最大并发连接数只有一条公式：
```nginx
最大连接数 = worker_processes × worker_connections
```
- `worker_connections` 是单个 worker 能同时打开的连接数上限。
- `worker_processes auto` 等于 CPU 核数，4 核就是 4 个 worker。
- 这个数是理论上限，不是保证值，见文末边界条件。

**算例 1：4 核、每 worker 1024 个连接，能接多少客户端？**
```text
静态直出：4 × 1024 = 4096 个连接，可接约 4096 个客户端
反向代理：4 × 1024 ÷ 2 = 2048 个客户端
```
为什么反向代理要除以 2？官方文档明确说，`worker_connections`
计入的是全部连接，不只是客户端连接，还包括到上游服务器的
连接。一个反向代理请求通常占 2 个连接：客户端—Nginx 一个，
Nginx—后端一个，所以能同时服务的客户端大约只有静态直出的一半。
换一组数验算：4 核、`worker_connections 4096` 时，最大连接数
为 4 × 4096 = 16384 个，反向代理客户端数为 16384 ÷ 2 = 8192 个。
想接更多客户端，先看瓶颈是不是这个公式。`worker_connections`
改大之前，先确认 `worker_rlimit_nofile` 和系统 `ulimit -n`
够不够：单个 worker 能打开的文件描述符数是硬上限，连接数
超过它，配置写得再大也接不进来。

**keys_zone 与 max_size：内存放 key，磁盘放内容，别配反了**
`proxy_cache_path` 里两个尺寸管的是两样东西：`keys_zone` 是
共享内存，只存缓存 key 和元数据，不存响应体；`max_size` 是
磁盘上限，存真正的缓存文件，超了由 cache manager 按最近
最少使用（LRU）淘汰。
官方文档给的换算口径是：开源版 1MB 的 keys_zone 大约能存
8000 个 key，相当于每个 key 约 131 字节；商业订阅版因存了
扩展信息，1MB 约 4000 个 key，要按一半算。

**算例 2：原文的 `keys_zone=api:50m max_size=1g` 能放多少？**
```text
10m keys_zone：10 × 8000 = 80000 个 key，约 8 万个
50m keys_zone：50 × 8000 = 400000 个 key，约 40 万个
1GB keys_zone：1024 × 8000 = 8192000 个 key，约 819 万个
```
再看磁盘那头。`max_size=1g` 能放几个文件，取决于平均响应大小：
平均 20KB 约 52428 个文件，平均 10KB 约 104857 个文件，
平均 2KB 约 524288 个文件。
配比就按这两头对齐。平均 20KB 时，52428 个文件只需
52428 ÷ 8000 ≈ 6.6MB 的 keys_zone，配 10m 绰绰有余，原文的
50m 更是留了数倍余量。但平均 2KB 时，524288 个文件需要
524288 ÷ 8000 ≈ 65.5MB 的 keys_zone，50m 反而不够：磁盘还没满，
key 先被 LRU 挤掉，缓存命中率就上不去。所以小文件多的接口要
先算 key 数，再定 keys_zone，不能只看 max_size 给了 1g 就放心。

**缓存命中率与回源量：90% 到 99% 不是只差 9 个点**
回源量 = 总请求量 ×（1 − 命中率）。命中率每提一点，回源量
是按倍数往下掉的，不是按百分点。

**算例 3：一天 100 万次请求，命中率 90% 与 99% 差多少？**
```text
命中率 90%：1000000 × 10% = 100000 次回源
命中率 99%：1000000 × 1% = 10000 次回源，相差 10 倍
```
换成每秒的数也一样：1000r/s 的接口，90% 命中时回源 100r/s，
99% 命中时回源只有 10r/s，后端压力直接降到原来的十分之一。
所以“命中率从 90% 提到 99%”真正的意思是“回源量砍掉 90%”，
而不是“只好了 9%”。命中率要拿 `$upstream_cache_status` 的
HIT 数除以 HIT 与 MISS 之和来算，别拿总请求数直接估。

**limit_req：rate 是漏速，burst 是桶深，nodelay 是不等**
原文配置 `rate=10r/s`、`burst=20`：漏速 10r/s，即每 100ms
漏出 1 个请求（1000 ÷ 10 = 100）。不加 `nodelay` 时，超速的
请求先排队，再按每 100ms 一个的速度放行：20 个突发请求排满队，
最后一个要等 20 ÷ 10 = 2.0 秒才被处理；第 22 个及以后的请求
（1 个即时处理 + 20 个排队名额已满）直接返回 503。
加上 `nodelay` 后，1 个即时处理 + 20 个 burst 名额 = 21 个请求
立即放行，第 22 个起同样返回 503。区别只在“等不等”，不在
“放多少”：nodelay 不提高持续处理能力，每个被立即放行的突发
请求仍占一个名额，名额要按 10r/s 的速度、每 100ms 回补一个，
20 个名额全部回满同样要 2.0 秒。

边界条件：
- 最大连接数公式算的是上限。长连接、keepalive、慢客户端都会让
  连接数先顶满，实际可服务客户端数低于算例。
- `worker_connections` 改大不等于并发一定上去。文件描述符上限、
  后端连接池、带宽、CPU 里的任何一个先顶满，都会先成为瓶颈。
- keys_zone 满了先淘汰最久未用的 key，max_size 满了先淘汰最久
  未用的文件。两头都可能先满，配比要按平均响应大小分别算。
- 命中率换算只对可缓存请求成立。`proxy_cache_valid` 没覆盖的
  状态码、带 `Authorization` 头又没显式区分 key 的请求不进缓存。
- `burst` 是按 `limit_req_zone` 的 key 算的，通常是单个 IP：20 个
  burst 是每个 IP 各有 20 个，不是全站一共 20 个。

## 小技巧与冷知识

【小技巧】`proxy_pass` 尾巴的斜杠决定 URI 怎么拼，别靠猜：`location /api/ { proxy_pass http://backend/; }` 带斜杠时，location 匹配到的 `/api/` 部分会被替换掉，请求 `/api/user` 转给后端的是 `/user`；不带斜杠 `proxy_pass http://backend;` 时，完整原始 URI 原样转发，后端收到的是 `/api/user`。原文只说了“拼接规则不一样”，落到实处就是“带斜杠替换匹配段、不带斜杠全量转发”，复制配置先看斜杠在不在。

【小技巧】`add_header` 在 location 里一写，server 层的同名配置就全丢了：Nginx 的 `add_header` 是按层级整体继承的，location 里只要写了一条 `add_header`，server 层写的所有 `add_header` 都不再生效，不会逐条合并。静态资源 location 里加了 `Cache-Control`，结果安全头 `X-Frame-Options` 悄悄没了，多半就是这个原因；要么在 location 里把需要的头全写一遍，要么改用 `more_set_headers` 这类第三方模块。

【小技巧】location 里的 `if` 尽量别用，改用 `map`：`if` 在 location 上下文里行为特殊，官方文档明确说明只有 `return` 和 `rewrite` 在 `if` 里是安全的，写 `proxy_pass`、`add_header` 这类指令可能出现预期外的结果。需要按条件分流时，先在 http 层用 `map $http_user_agent $backend { ... }` 把条件算成一个变量，再在 location 里按变量值正常写指令；条件判断和转发逻辑分开，配置才不会时灵时不灵。

【小技巧】`proxy_pass` 写死域名时，IP 是启动那一刻解析一次的：`proxy_pass http://api-service:8080;` 里域名只在 Nginx 启动或 reload 时解析，后端容器 IP 变了（比如 Docker 里容器重建）Nginx 还会往旧 IP 发。改成带变量的写法 `set $upstream api-service:8080; proxy_pass http://$upstream;`，并配 `resolver 127.0.0.11 valid=30s;`，Nginx 才会在每次请求时重新解析；容器化部署里后端地址常变，这一行是刚需。

【小技巧】想让 Nginx 到上游的连接复用，别只写 `keepalive`：upstream 块里写 `keepalive 32;` 只是开了连接池，还要配 `proxy_http_version 1.1;` 和 `proxy_set_header Connection "";`，否则 Nginx 默认用 HTTP/1.0 回源、每次请求都新建连接。原文的 WebSocket 一节用了 1.1 和 Upgrade 头，那是升级协议；普通回源复用连接要的是把 Connection 头清空，三行缺一不可，少一行连接池就形同虚设。

【小技巧】后端挂了先别急着 502，用 `proxy_cache_use_stale` 顶一会儿：`proxy_cache_use_stale error timeout updating;` 让后端报错、超时或正在更新缓存时，Nginx 先把缓存里过期的旧内容顶上去，用户看到的是几分钟前的数据而不是错误页。再配 `proxy_cache_background_update on;`，过期后由 Nginx 在后台悄悄回源刷新，前台请求全程不被拖慢；原文的 proxy_cache 只讲了命中与否，这两行是“缓存当降级兜底”的用法。

【小技巧】写重定向和日志时分清 `$uri` 和 `$request_uri`：`$uri` 是经过解码和规范化后的路径（`%20` 变回空格、`/../` 被折叠掉），`$request_uri` 是客户端发来的原始完整 URI（带查询串、原样不动）。`return 301 https://$host$request_uri;` 用后者才能原样保留查询参数；用 `$uri` 拼跳转，带中文或特殊字符的地址会被改写一次，排障时先看你用的是哪一个变量。

【小技巧】鉴权别在每个 location 里重复写逻辑，用 `auth_request`：`location /api/ { auth_request /auth; proxy_pass http://backend; }` 配一个 `location = /auth { internal; proxy_pass http://auth-service/verify; }`，Nginx 会先发一个子请求问鉴权服务，返回 2xx 才放行、401/403 直接拦。鉴权逻辑集中在一个服务里，业务接口不用再各自校验 token；原文的限流和缓存都是 Nginx 自带能力，这招是把“先问一嘴再放行”也收进 Nginx。

【冷知识】Nginx 的名字读作 engine-x，作者是俄罗斯工程师 Igor Sysoev：他 2002 年开始写 Nginx，第一个公开版本在 2004 年 10 月发布，最初是为了解决 C10K 问题——一台服务器同时撑住一万个并发连接。名字里的 X 就是“未知数 / 扩展”的意思，读法不是字母 N-G-I-N-X，而是 engine 加 x。

【小技巧】怀疑某个模块没编译进去，先跑 `nginx -V`：大写的 `-V` 会打印编译参数，包括 `--with-http_ssl_module`、`--with-stream` 这类模块开关。配 `stream` 做四层代理或配 `http2` 报错“未知指令”，先看 `-V` 输出里有没有对应模块，没有就说明是编译时没加、不是配置写错了；小写的 `-v` 只打印版本号，查模块必须用大写。

## 实战案例

### 案例一：静态资源突然 404，配置“看着都对”，请求走到了错误的 location

以下为设计演练示例，非真实生产事故。

**现象**

前端新版上线后，`app.js` 与 `app.css` 请求返回 404，后端接口正常；直接访问后端端口能拿到文件，走 Nginx 就 404。配置里静态 location 与反代 location 看着都在，改完也 `nginx -t` 通过了。

**排查**

先用 `nginx -T` 打印全部生效配置（含被 include 进来的），逐条对实际生效的 location，而不是只看自己改的那个文件。再按优先级表逐条核对：`=` 精确匹配 > `^~` 前缀匹配 > 正则（按文件顺序）> 普通前缀。常见真相有两种：一是某个 `^~` 前缀把静态请求截胡了，正则 location 根本没机会匹配；二是 `root` 与 `alias` 混用——location 里的 root 拼接的是完整请求路径，例如 `root /data;` 加请求 `/img/a.png`，找的是 `/data/img/a.png`，想把 `/img/` 这段去掉要改用 alias，别硬试 root。最后看 error_log 的最后几行（排障先看 error_log，access_log 只记录请求结果）。

**根因**

location 的裁决按优先级表，不按书写顺序；`root` 拼完整 URI、`alias` 替换匹配段，两者拼接规则不同。静态 location 写了正则 `~* \.(js|css|png|jpg)$`，前面又有一条 `^~ /static/` 或 `^~ /` 之类的前缀，请求还没走到正则就被截胡；或者 root 写错了层级，Nginx 在磁盘上找的路径根本不存在。

**修复**

精确匹配用 `=`，前缀想截胡用 `^~`，正则注意先后顺序；静态资源后缀匹配用不区分大小写的 `~*`（原文动静分离一节）。root/alias 二选一写对：`root` 拼完整路径，`alias` 替换匹配段。改完固定动作是 `nginx -t` 通过后再 `nginx -s reload`，不要直接 restart。修完用 `curl` 带 Host 头逐条验证静态路径与反代路径，确认 404 消失、后端收到的还是正确 URI（`proxy_pass` 尾巴的斜杠决定 URI 怎么拼，复制配置先看斜杠在不在）。

**复盘**

- 排障顺序：`nginx -T` 看生效配置 → 按优先级表核对 → 看 error_log 的最后几行；跳过前两步直接改配置，修完还是错的。
- `^~` 是截胡不是“优先级高一点”：一旦命中，正则连看都不看。
- `server_name` 写了域名但用 IP 测试也会落到默认 server：按域名选 server 靠的是请求里的 Host 头，浏览器输 IP 时 Host 就是 IP，自然落到默认 server；用域名测试，或手动带 Host 头验证。

### 案例二：正常请求被 503 限流，缓存又一直 MISS，两个配置各自“没错”

以下为设计演练示例，非真实生产事故。

**现象**

高峰期部分正常用户的请求返回 503，同时排行榜接口的 `X-Cache-Status` 一直是 MISS，后端压力没降下来。限流配置与缓存配置分开看都“符合原文”，合起来就出问题。

**排查**

先分清 503 是谁返回的：限流的 503 来自 `limit_req`，后端挂了的 502/503 来自上游。按定量节的限流口径算：原文配置 `rate=10r/s`、`burst=20`，漏速 10r/s 即每 100ms 漏出 1 个请求；不加 `nodelay` 时超速请求先排队，20 个突发请求排满队，最后一个要等 20 ÷ 10 = 2.0 秒才被处理，第 22 个及以后的请求（1 个即时处理 + 20 个排队名额已满）直接返回 503。加上 `nodelay` 后 1 个即时处理 + 20 个 burst 名额 = 21 个请求立即放行，第 22 个起同样返回 503。再看缓存：`keys_zone` 够不够要先算 key 数再定——平均 2KB 的小文件接口，524288 个文件需要约 65.5MB 的 keys_zone，原文的 50m 反而不够，磁盘还没满，key 先被 LRU 挤掉，命中率就上不去。

**根因**

两处都是“配比”问题，不是“开关”问题：`burst` 是按 `limit_req_zone` 的 key 算的，通常是单个 IP——20 个 burst 是每个 IP 各有 20 个，不是全站一共 20 个；只看“全站才 20 个”会误判限流太严，实际是某个 IP 的突发超过了 20 个名额。缓存那头是 keys_zone 与 max_size 没按平均响应大小对齐：小文件多的接口只看 max_size 给了 1g 就放心，key 数先顶满，缓存一直 MISS。

**修复**

限流侧：确认超速的是单个 IP 还是全站——是单个 IP 的突发，调大该 zone 的 burst 或给正常调用方单独的 zone；是全站流量上涨，调 rate 而不是只调 burst（只调 r 不调 B 是治标的另一面：nodelay 不提高持续处理能力，每个被立即放行的突发请求仍占一个名额，名额要按 10r/s 的速度、每 100ms 回补一个，20 个名额全部回满同样要 2.0 秒）。缓存侧：先算 key 数再定 keys_zone，平均响应越小、keys_zone 反而要越大；带 `Authorization` 头的接口默认不缓存，确认排行榜接口没带用户态头、或已显式在 `proxy_cache_key` 里带上区分维度。修完看 `X-Cache-Status` 的 HIT 数除以 HIT 与 MISS 之和算命中率，别拿总请求数直接估。

**复盘**

- 限流排障先算漏速与队尾等待（每 100ms 漏 1 个、队尾等 2.0 秒、第 22 个起 503），再确认 burst 是按 key（单个 IP）算的；缓存排障先算 key 数与 keys_zone 的配比，再看 X-Cache-Status。
- `nodelay` 的区别只在“等不等”，不在“放多少”：21 个立即放行与 1 个立即放行 + 20 个排队，放行总量一样。
- 命中率换算只对可缓存请求成立：`proxy_cache_valid` 没覆盖的状态码、带 `Authorization` 头又没显式区分 key 的请求不进缓存，先确认请求可缓存再谈命中率。

## 面试与自测

1. Nginx 的 location 匹配顺序是怎样的？`^~` 在其中起什么作用？
2. 改完 Nginx 配置后，为什么标准动作是先 `nginx -t` 再 `nginx -s reload`，而不是直接 restart？
3. 后端日志里拿到的客户端 IP 全是 127.0.0.1，通常是漏了什么？

4. 一台 4 核服务器配 worker_processes 4、worker_connections 1024，反向代理场景下最多能同时服务多少客户端？如果把缓存命中率从 90% 提到 99%，回源量会变成原来的几分之几？

### 参考答案

1. 顺序是 `=` 精确匹配 > `^~` 前缀匹配（不再看正则）> `~` / `~*` 正则（按文件顺序，区分/不区分大小写）> 普通前缀 / 通用 `/`。想让某个前缀截胡正则时用 `^~`；排障时以 `nginx -T` 打印出的全部生效配置为准逐条对。
2. 因为 restart 是完全重启，会断连接，且语法有错时服务会起不来直接宕机。固定动作是 `nginx -t` 只测试语法不动正在跑的服务，通过后再 `nginx -s reload` 平滑重载：master 让新 worker 用新配置、旧 worker 处理完存量连接再退出，所以 reload 不断连接。顺序不能反。
3. 漏了反向代理的 `proxy_set_header` 三件套：`proxy_set_header Host $host;`、`proxy_set_header X-Real-IP $remote_addr;`、`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。不写这三行，后端拿到的都是 127.0.0.1，日志和限流全错。

4. 4×1024=4096 个连接，反向代理每客户端占 2 个，只剩 2048 个；命中率 90% 回源 10%，99% 回源 1%，回源量变为原来的十分之一。
