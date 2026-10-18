# Docker

> 每晚 20:30 自动补充完善。贴合本地开发（Windows + Docker Desktop）和项目交付场景。

## 常用命令速查

```bash
# 容器
docker ps -a
docker run -d --name redis -p 6379:6379 redis:7
docker exec -it <容器> bash        # 进容器
docker logs -f --tail 100 <容器>   # 看日志
docker stop/start/restart/rm <容器>
docker cp 本地路径 容器:容器路径    # 拷文件

# 镜像
docker images
docker pull nginx:alpine
docker build -t myapp:1.0 .
docker rmi <镜像>
docker system prune -a             # 大扫除（删无用镜像/容器/网络/缓存）

# 信息
docker inspect <容器>              # 看挂载、IP、环境变量
docker stats                       # 实时资源占用
```

## Dockerfile 最佳实践

```dockerfile
# 多阶段构建：编译和运行分离，镜像从几百 MB 降到几十 MB
FROM maven:3.8-openjdk-8 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline      # 先下依赖，利用层缓存
COPY src ./src
RUN mvn package -DskipTests

FROM openjdk:8-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

要点：
1. **层缓存**：变化少的文件（pom.xml）先 COPY，变化多的（src）后 COPY。
2. **alpine 镜像**：体积小，但注意 glibc 兼容问题（需要装 `gcompat` 或换 slim 版）。
3. **.dockerignore**：把 `target/`、`.git`、`*.md` 排除，别打进镜像。
4. **时区**：Java 容器加 `-e TZ=Asia/Shanghai`，不然日志时间是 UTC。
5. **JVM 内存**：容器里设 `-Xmx` 别超过容器 limit；JDK 8u191+ 才识别 cgroup 内存限制，老版本会按宿主机内存算直接 OOM。

## docker-compose 常用

```yaml
version: "3.8"
services:
  mysql:
    image: mysql:5.7
    ports: ["3306:3306"]
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - mysql-data:/var/lib/mysql
  redis:
    image: redis:7
    ports: ["6379:6379"]
volumes:
  mysql-data:
```
```bash
docker compose up -d
docker compose down        # 停+删容器（volume 默认保留）
docker compose down -v    # 连数据卷一起删，慎用
```

## 网络与数据卷

- 默认 bridge 网络，容器互访用**服务名**（compose 里）或 `--link`（已过时，别用）。
- 数据要持久化必须挂 volume，容器删了数据还在；`docker volume ls` 查看。
- 调试网络：`docker network inspect <网络名>` 看容器 IP。

## Windows 下注意点

1. Docker Desktop 吃内存：Settings → Resources 里限制内存/CPU，别让它拖慢 Ollama。
2. 文件挂载慢：代码放 WSL2 文件系统里（`\\wsl$`），别放 Windows 盘，IO 差几倍。
3. 换行符：Windows 的 CRLF 进 Linux 容器跑 shell 脚本会炸，`.gitattributes` 或编辑器统一 LF。

## 常见问题

1. **端口被占**：`docker ps` 看谁占了，或 `netstat -ano | findstr 8080`（Windows）。
2. **容器起不来**：先 `docker logs`，90% 的问题日志里都有。
3. **镜像拉不动**：换国内镜像源（daemon.json 配 `registry-mirrors`）。
4. **磁盘爆了**：`docker system df` 看占用，`docker system prune -a` 清理。
5. **容器内连不上宿主机**：用 `host.docker.internal`（Docker Desktop 支持）。

## 待补充
- 本地 Ollama 进阶：不用 Docker，直接本机跑（已在用）


## 疑难杂症

### 1. 容器内时区不对，日志和定时任务全错乱
- **现象**：容器里 `date` 显示 UTC，日志时间对不上，cron 定时任务跑错时间。
- **原因**：官方镜像默认 UTC 时区，和宿主机不一致。
- **解决**：挂载时区文件 `-v /etc/localtime:/etc/localtime:ro`，或 Dockerfile 里 `ENV TZ=Asia/Shanghai` + 装 tzdata。

### 2. 多阶段构建缓存每次都失效
- **现象**：改一行代码，`docker build` 全量重跑，慢得离谱。
- **原因**：COPY 顺序不对，源代码变动导致后面所有层缓存失效。
- **解决**：先 COPY 依赖描述文件（package.json/pom.xml）并安装依赖，再 COPY 源码；`.dockerignore` 排除无关文件。

### 3. Windows 下文件挂载又慢又报权限错
- **现象**：Docker Desktop 挂载 Windows 目录，读写巨慢，偶发 permission denied。
- **原因**：跨 OS 文件共享走网络文件系统，性能差且权限映射复杂。
- **解决**：代码放 WSL2 文件系统里再挂载；或尽量用 COPY 进镜像而不是挂载；生产环境用 Linux 宿主机。

## 生产发布：tag 规范与滚动更新

```bash
# tag 规范：git short sha，永远别用 latest 当生产 tag
docker build -t registry/app:$(git rev-parse --short HEAD) .
docker push registry/app:abc1234

# compose 滚动更新
docker compose pull && docker compose up -d --no-deps app
```

- `latest` 是玄学 tag，多机部署时各机器 pull 到的可能不是同一版，出问题查都查不清。
- 回滚就是 `up -d` 上一个 tag，5 秒完成，所以每个 tag 都要能独立跑（数据库 migration 要向后兼容）。
- 健康检查写上（`healthcheck`），compose 等容器 healthy 才切流量，不然刚启动就接请求直接 500。

## 多阶段构建实战（Java 版）

```dockerfile
# 阶段1：构建
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline          # 先下依赖，利用缓存
COPY src ./src
RUN mvn package -DskipTests
# 阶段2：运行（只要 JRE + jar，镜像从 800MB 压到 200MB）
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

- `COPY pom.xml` 先于 `COPY src`：依赖层缓存，只有改 pom 才重下依赖。
- Python 版同理：`python:3.12-slim` 做运行阶段，`pip install` 放构建阶段。
