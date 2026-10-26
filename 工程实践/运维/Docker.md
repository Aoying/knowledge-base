---
知识点: Docker
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - 工程实践知识点地图
  - Nginx
标签:
  - 运维
  - 实践
---

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

## Dockerfile 安全：别用 root 跑

```dockerfile
FROM eclipse-temurin:17-jre
RUN addgroup --system app && adduser --system --ingroup app app
WORKDIR /app
COPY --chown=app:app app.jar .
USER app
ENTRYPOINT ["java","-jar","app.jar"]
```

- 容器逃逸的新闻年年有，非 root 是最低成本的防线。
- `COPY --chown` 一步到位，别 `RUN chown`（多一层还慢）。
- 只读文件系统更进一步：`docker run --read-only`，写操作走 volume。

## docker-compose 生产示例

```yaml
services:
  app:
    image: registry/app:${TAG:-latest}
    deploy:
      resources: { limits: { cpus: '2', memory: 2G } }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s; retries: 3
    restart: unless-stopped
    logging: { driver: json-file, options: { max-size: 100m, max-file: "3" } }
```

- `resources.limits` 必加，不加的话一个容器 OOM 能把宿主机拖死。
- 日志 `max-size` 必加，不然 json-file 无限涨（见"磁盘爆了"疑难杂症）。
- `${TAG:-latest}` 让 CI 传 tag，本地默认 latest。

## 基础知识补强

### 核心概念与术语

- **容器（Container）**：把应用和它依赖的环境打包在一起运行的隔离进程。本质是宿主机上的一个进程，只是被隔离了视图和资源，不是虚拟机。
- **容器 vs 虚拟机**：虚拟机虚拟出一整套硬件并运行完整操作系统，启动以分钟计；容器共享宿主机内核，只隔离进程、文件系统和网络，启动以秒计，占用小得多。
- **镜像（Image）**：容器的模板，只读、不可变。打包了应用代码、运行时、依赖库和默认配置。容器是镜像运行起来的实例：一个镜像可以跑出很多个容器。
- **仓库（Registry）**：存放和分发镜像的地方。官方公共仓库是 Docker Hub；公司内部一般有私有仓库。镜像名格式：`仓库地址/命名空间/镜像名:标签`。
- **标签（Tag）**：镜像的版本标记，如 `redis:7`、`nginx:alpine`。不写标签默认是 `latest`，它只是个普通标签，不保证是最新版。
- **Dockerfile**：构建镜像的脚本，一条指令生成一层文件系统变更，按顺序执行。
- **镜像分层（Layer）**：镜像由多层只读层叠加而成，多个镜像可以共享相同的基础层，省磁盘也省拉取时间。容器运行时在最上面加一层可写层，容器内改文件只动这一层。
- **数据卷（Volume）**：由 Docker 管理的持久化存储，独立于容器生命周期，容器删了卷还在。
- **守护进程与客户端**：`docker` 命令是客户端，真正干活的是后台守护进程（daemon）。命令报错先分清是客户端问题还是守护进程没起来。
- **容器 ID 与名称**：每个容器有唯一 ID（一长串哈希），命令里可以用 ID 的前几位代替全串；名称是在 ID 之外给人用的别名，二者都可用来指定容器。
- **构建上下文（Build Context）**：执行 `docker build` 的目录就是上下文，Dockerfile 里的 COPY 只能拷贝这个目录范围内的文件，范围外拷不到。上下文越大构建越慢，这也是要配忽略清单的原因。
- **重启策略（Restart Policy）**：容器退出后要不要自动拉起的规则。`no` 是不拉（默认），`always` 是总是拉，`unless-stopped` 是除非人为停过都拉。长期跑的服务一般选后两者，免得宿主机重启后服务全趴。

### 基础语法与用法

`docker run` 一个参数一个参数拆开看：

```bash
docker run -d --name myredis -p 6380:6379 -e TZ=Asia/Shanghai -v redis-data:/data redis:7
```

- `-d`：后台运行（detached），不加则前台占用当前终端，终端关了容器就退。
- `--name`：给容器起名字，后续命令用名字代替一长串容器 ID，好记必加。
- `-p 6380:6379`：端口映射，冒号前是宿主机端口，后是容器内端口。只写容器端口则宿主机随机分配。
- `-e KEY=VALUE`：给容器传环境变量，应用读配置最常用的方式。
- `-v 卷名:容器内路径`：挂载数据卷到容器内目录，数据写在这里才不丢。
- 最后一截 `redis:7` 是镜像名，镜像后面跟的是传给容器内程序的命令（可选）。

容器生命周期常用操作：

```bash
docker run -d --name demo nginx        # 创建并启动
docker ps                              # 只看运行中的；-a 看全部（含已停止）
docker stop demo                       # 优雅停止：先发终止信号，等进程收尾
docker start demo                      # 启动已停止的容器（配置和数据都还在）
docker restart demo                    # stop + start
docker rm demo                         # 删除已停止的容器；运行中的要先 stop
docker rm -f demo                      # 强制删运行中的容器，慎用：等于直接断电
docker logs demo                       # 看容器主进程的标准输出，不是进容器翻文件
docker exec -it demo sh                # 在运行中的容器里开个 shell 排查问题
docker run --rm -it alpine sh          # 临时容器：退出即自动删除，试命令用
```

构建镜像和管数据卷的最小命令组：

```bash
docker build -t myapp:1.0 .        # 用当前目录的 Dockerfile 构建，-t 给名和标签
docker build -f Dockerfile.dev .   # 指定用哪个 Dockerfile
docker volume ls                   # 看有哪些数据卷
docker volume create mydata        # 手动建卷（不建也行，run 时自动建）
docker rmi myapp:1.0               # 删镜像；还有容器在用它时删不掉
```

- `docker build` 最后的那个点不是笔误，它就是构建上下文路径，漏了命令直接报错。
- 镜像删不掉先查是不是有停止状态的容器还引用着它，先删容器再删镜像，顺序别反。

重新理解几个高频命令：

- `docker pull 镜像`：只下载镜像，不运行。`run` 时本地没有镜像会自动先 pull。
- `docker images` 看的是镜像（模板），`docker ps` 看的是容器（实例），两者别混。
- `docker inspect 容器` 输出完整 JSON：查它实际用的镜像、挂载、网络、环境变量，以这个为准，别靠记忆。

### 关键机制速览

- **容器为什么会自己退出**：容器活着的前提是它的 1 号进程（主进程）活着。主进程结束，容器立即停止。所以让 Web 服务当前台进程跑是对的，用后台方式启动应用容器会秒退。
- **CMD 与 ENTRYPOINT 的区别**：都在 Dockerfile 里定义容器启动时跑什么。`ENTRYPOINT` 是固定入口（程序本体），`CMD` 是默认参数或默认命令；`docker run 镜像 参数` 会覆盖 CMD，但不会覆盖 ENTRYPOINT。简单项目只写 CMD 就够。
- **端口映射的心智模型**：容器有自己独立的网络空间，容器内的 8080 和宿主机的 8080 是两回事。不做 `-p` 映射，外部就访问不到容器里的服务（同一 Docker 网络内的其他容器除外，它们用容器端口互访）。
- **三种挂载方式**：
  - 命名卷（volume）：Docker 管理，适合存数据库数据，正规做法。
  - 绑定挂载（bind mount）：把宿主机某个目录直接映射进容器，适合本地开发时挂代码，改完立即生效。
  - 容器可写层：不挂载直接写。容器一删数据全没，只适合临时文件。
- **镜像构建缓存**：Dockerfile 每条指令对应一层，构建时从上往下比对，没变的层直接用缓存；某一层变了，它之后的层全部重建。这就是依赖文件要先于源码拷贝的原理。
- **compose 的定位**：单容器用 `run`，多容器（一套服务加数据库加缓存）用 compose 一个文件声明，`up` 一键拉起同一网络内的全部服务，服务间用服务名互相访问。
- **默认网络的心智模型**：不指定网络时容器都接在默认的桥接网络上，有各自的内网 IP，可以出网，但默认不能用容器名互相解析。compose 会为一套服务自建网络，在自建网络里容器名就是可解析的主机名，这就是 compose 里能用服务名互访的原因。
- **重启策略何时生效**：策略在容器退出时触发判断。手动 `docker stop` 停掉的容器在 `always` 下不会被立刻拉起（视为人为操作），但宿主机或守护进程重启后会被拉起；`unless-stopped` 则连这种重启也不拉。测试时拿它当自动复活开关会误判。

### 新手易错点

- **把容器当虚拟机用**：在容器里装一堆服务、手动改配置、指望重启还在——全错。容器应该无状态、可随时删了重建；配置靠环境变量和挂载注入，数据靠卷。
- **数据没挂卷就删容器**：数据库容器没挂卷时，数据在可写层里，`docker rm` 之后数据不可恢复。先挂卷，再谈删除。
- **端口映射写反**：`-p 6379:6380` 把宿主机 6379 映射到容器 6380，服务监听的是 6379，当然连不上。记住：左宿主机，右容器。
- **用 latest 上生产**：`latest` 指向会漂移，今天拉的和下个月拉的可能不是同一个镜像。出问题先把标签固定到具体版本。
- **`docker stop` 后以为数据丢了**：stop 只是停进程，容器还在，`docker start` 就能回来；数据丢失发生在 `rm` 且没挂卷时。
- **在容器里改了文件当修复**：`docker exec` 进去改配置当时有效，容器重建后改动消失。正解是改 Dockerfile 或挂载的配置文件后重建。
- **清理命令乱用**：`docker system prune -a` 会删掉所有未被容器使用的镜像和缓存，机器上其他项目的镜像也会没，下次用要重新拉。执行前先用 `docker system df` 看清要清什么，只清确定不要的。
- **分不清删的是镜像还是容器**：`rm` 删容器、`rmi` 删镜像，敲混了要么报不存在，要么删错对象。删之前分别用 `docker ps -a` 和 `docker images` 各看一眼当前列表。
