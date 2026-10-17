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
- 生产发布：镜像 tag 规范、滚动更新
