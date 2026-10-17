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
