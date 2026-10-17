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

- HTTPS / 证书自动续期（certbot）
- 限流（limit_req）防刷配置

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
