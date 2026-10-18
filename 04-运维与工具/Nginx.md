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

## 待补充



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
