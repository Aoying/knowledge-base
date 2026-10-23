---
知识点: Maven与依赖管理
层级: 基础
适用版本: Maven 3.x（通用）
更新日期: 2026-10-23
关联:
  - CI/CD
  - Docker
  - Spring-Boot
  - 单元测试与Mock
标签:
  - 运维
  - 实践
  - 速查
---

# Maven 与依赖管理

> Maven 做两件事：**按约定组织构建生命周期**，**按坐标管理依赖图**。翻车大多不在编译，而在依赖图——同一库两个版本共存、传递进来不想要的版本、私服拉不到包。与 [CI/CD](工程实践/CICD.md) 篇的分工：那篇讲流水线整体，本篇讲 `mvn` 这一步内部发生了什么。

## 1. 核心概念

- **坐标（GAV）**：`groupId:artifactId:version` 唯一确定一个构件，仓库里的物理路径就是坐标的展开。
- **仓库分三层**：本地仓库（`~/.m2/repository`，缓存）→ 私服（内网 Nexus/Artifactory 一类，代理中央仓库 + 托管自研包）→ 中央仓库。查找先本地、再私服、最后才出网。
- **pom 是声明式模型**：`dependencies` 只声明「要什么坐标」，实际拿到哪个版本由调解规则决定（第 3 节）——这是「我明明写的是 A 版本，跑起来却是 B」的根源。

## 2. 基础知识：生命周期与常用命令

三套生命周期互相独立：**clean**（清理）、**default**（构建部署，主线）、**site**（站点文档）。default 的阶段按序排列（compile → test → package → verify → install → deploy），执行某个 phase 时它之前的 phase 一起执行；分工上 surefire 在 test 阶段跑单元测试、failsafe 在 verify 阶段跑集成测试，install 把产物装进本地仓库，deploy 才推私服。

```bash
mvn clean install               # 全量构建并装本地仓库（多模块在根目录执行）
mvn clean package -DskipTests   # 打包不跑测试（救急可用，别成习惯）
mvn test -Dtest=FooServiceTest  # 只跑指定测试类（排错常用）
mvn -pl module-a -am package    # 只构建 module-a 及它依赖的上游模块（第 4 节）
mvn -o package                  # 离线模式，验证本地缓存是否完整
```

跳过测试分两档：`-DskipTests`（编译但不执行）与 `-Dmaven.test.skip=true`（连编译都跳）。**Wrapper（mvnw）**把 Maven 版本随项目走，流水线里优先用 `./mvnw` 而非机器自带的 `mvn`，避免版本不一致翻车。

## 3. 进阶知识：依赖范围与传递规则

### scope（依赖范围）

| scope | 生效范围 | 打进产物 | 典型例子 |
|---|---|---|---|
| compile（默认） | 编译/测试/运行 | ✓ | Spring、MyBatis |
| provided | 编译/测试 | ✗ | servlet-api（容器提供） |
| runtime | 测试/运行 | ✓ | 数据库驱动、日志实现 |
| test | 仅测试 | ✗ | JUnit、Mockito |
| system | 指向本地文件路径 | — | **不要用**，见第 6 节 |
| import | 仅 dependencyManagement 导入 BOM | — | 见下文 |

### 传递依赖的三条调解规则

1. **最近定义优先**：离根越近的版本胜出。A→B→C(1.0) 与 A→C(2.0) 冲突时，深度更浅的 2.0 赢（与 Gradle 的「最高版本胜出」不同，跨生态时容易记混）。
2. **先声明优先**：深度相同时，先声明的路径胜出，声明顺序不是随便写的。
3. **可选与排除**：`<optional>true</optional>` 让依赖不向外传递；`<exclusions>` 在引入点剪掉传递路径。两者是防御手段，治本靠版本统一。

### dependencyManagement 与 BOM

`dependencyManagement` 只声明版本、**不实际引入**；各模块在 `dependencies` 里只写坐标不写版本，版本以它为准，父 pom 统一全工程版本靠这一节。**BOM** 是只含 dependencyManagement 的 pom，用 `scope=import` 导入——Spring Boot 的 `spring-boot-dependencies` 即 BOM，导入后它管理的那批库全不用写版本。注意：`dependencies` 里显式写死的版本会盖过 dependencyManagement，「统一不生效」先搜散落的硬编码版本。

## 4. 实战应用：冲突排查与多模块

### 冲突排查流程

症状多在运行时才爆：`NoSuchMethodError`、`ClassNotFoundException`——编译能过、跑起来缺方法，九成是类被加载成了错误版本。

```bash
mvn dependency:tree                       # 全量依赖树，看目标库有几个版本、从哪进来
mvn dependency:tree -Dincludes=组id:构件id  # 只看某个库的路径（排查主力）
mvn dependency:tree -Dverbose              # 连被调解掉的版本也显示（omitted 标注）
```

处理顺序：定位期望版本应走的路径 → 引入点 `exclusions` 剪掉错误路径，或在 dependencyManagement 钉死版本 → 重跑 `dependency:tree` 确认只剩一个版本 → 跑测试验证。团队项目可加 `maven-enforcer-plugin` 的依赖收敛规则，让冲突在构建期报错而非运行时。

### 多模块：聚合与继承是两件事

- **聚合**：父 pom 用 `<modules>` 组织一次构建，反应堆（reactor）按模块间依赖自动排序，解决「构建顺序」。
- **继承**：子 pom 用 `<parent>` 继承 dependencyManagement 与插件配置，解决「配置复用」。
- 两者常同时出现但概念独立。部分构建用 `-pl` 选模块、`-am` 连带上游、`-amd` 连带下游；发布前仍要在根目录全量跑一遍。

## 5. 私服与 settings.xml

- **私服的双重角色**：代理缓存中央仓库（内网出不了外网、直连慢）+ 托管自研公共模块的分发。
- **settings.xml**（`~/.m2/settings.xml`，不进版本库）：配镜像（mirror 把 central 流量导向私服）与凭据（`servers` 节的 id 要与 pom/镜像配置对应）。凭据只放这里或 CI 的 Secrets，**绝不写进 pom**。
- **SNAPSHOT 与 RELEASE**：SNAPSHOT 可变，Maven 默认按更新策略定期检查（`-U` 可强制刷新）；RELEASE 发布后不可覆盖。「同事更新了公共包我却没生效」先确认是不是 SNAPSHOT，再试 `-U`。高频混淆：`install` 只装进**自己**的本地仓库，`deploy` 才推私服——只 install，同事永远拉不到。

## 6. 疑难杂症与常见误区

- **依赖下不下来**：按序查——settings.xml 镜像地址 → 私服上该版本是否真实存在 → 本地仓库是否有下载失败的残留（`*.lastUpdated` 是失败标记，删掉对应目录重试）→ 网络/代理。
- **本地能构建、CI 挂在依赖上**：CI 是干净环境。用 `-Dmaven.repo.local=/tmp/m2test` 换空仓库跑一遍，能复现就是某个包只存在于你的本地（忘了 deploy 或私服上没有）。
- **误区**：用 system scope 或把 jar 拷进项目（绕过依赖图，CI 立刻不一致；闭源 jar 应入私服）；用 LATEST/RELEASE 当版本（构建随时间漂移、不可复现）；在每个子模块重复声明同一版本（升级漏改一处即冲突源）。

## 7. 性能优化与版本差异

- 并行构建 `mvn -T 1C`：模块无依赖时并行，个别老插件线程不安全会翻车，遇问题退回串行定位；缩小范围（`-pl/-am`）通常比提速更有效。
- 3.8.x 起默认屏蔽不安全的 HTTP 仓库（http blocker）：老私服若仍是 http 地址，升级 Maven 后会突然全线下不动包，先查这条。3.9.x 为当前主流稳定线，插件兼容性以各插件官方说明为准。

## 8. 面试与自测

1. Maven 依赖调解规则是什么？深度相同冲突时谁胜出？
2. `dependencyManagement` 与 `dependencies` 的区别？BOM 如何导入？
3. 运行时报 `NoSuchMethodError` 但编译通过，排查步骤是什么？
4. install 与 deploy 的区别？SNAPSHOT 为什么同事更新了你却拉不到？
5. 聚合与继承有什么区别？

## 9. 关联知识

- 相关：[CI/CD](工程实践/CICD.md)（流水线中的构建阶段与依赖缓存）、[Docker](工程实践/Docker.md)（多阶段构建与层缓存）
- [Spring-Boot](开发语言/Java/Spring-Boot.md)：Boot 的 parent 与 BOM 如何接管版本管理
- [单元测试与Mock](工程实践/单元测试与Mock.md)：test scope 与 surefire/failsafe 分工的落点
- 延伸：Gradle 的版本调解（最高版本胜出）与版本目录是另一套思路，对照第 3 节理解差异
