---
知识点: Nginx
层级: 基础
适用版本: 通用
更新日期: 2026-10-30
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

## 面试与自测

1. Nginx 的 location 匹配顺序是怎样的？`^~` 在其中起什么作用？
2. 改完 Nginx 配置后，为什么标准动作是先 `nginx -t` 再 `nginx -s reload`，而不是直接 restart？
3. 后端日志里拿到的客户端 IP 全是 127.0.0.1，通常是漏了什么？

### 参考答案

1. 顺序是 `=` 精确匹配 > `^~` 前缀匹配（不再看正则）> `~` / `~*` 正则（按文件顺序，区分/不区分大小写）> 普通前缀 / 通用 `/`。想让某个前缀截胡正则时用 `^~`；排障时以 `nginx -T` 打印出的全部生效配置为准逐条对。
2. 因为 restart 是完全重启，会断连接，且语法有错时服务会起不来直接宕机。固定动作是 `nginx -t` 只测试语法不动正在跑的服务，通过后再 `nginx -s reload` 平滑重载：master 让新 worker 用新配置、旧 worker 处理完存量连接再退出，所以 reload 不断连接。顺序不能反。
3. 漏了反向代理的 `proxy_set_header` 三件套：`proxy_set_header Host $host;`、`proxy_set_header X-Real-IP $remote_addr;`、`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。不写这三行，后端拿到的都是 127.0.0.1，日志和限流全错。
