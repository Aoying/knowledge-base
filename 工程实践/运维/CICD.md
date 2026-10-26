---
知识点: CI/CD
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - Git-运维与工具
  - Docker
  - Nginx
标签:
  - 运维
  - 速查
  - 实践
---

# CI/CD

## 1. 核心概念

- **CI（持续集成）**：代码合入主干后自动完成构建、静态检查、单元测试，让问题在提交后几分钟内暴露，而不是上线前集中爆雷。
- **CD（持续交付/部署）**：流水线产出可发布的制品，自动部署到测试/预发环境；持续交付保留人工批准上线，持续部署则全自动直达生产。

一条流水线解决三件事：**可重复**（每次构建过程一致，不依赖某台机器的手工操作）、**可追溯**（哪个提交产出了哪个制品）、**可回滚**（发布失败能快速退回上一个好版本）。

## 2. 基础知识

### 流水线阶段

| 阶段 | 做什么 | 失败策略 |
|---|---|---|
| Checkout | 拉取代码、子模块 | 终止 |
| Build | 编译/打包（Maven、npm、Docker build） | 终止 |
| Test | 单元测试、集成测试、覆盖率 | 终止并出报告 |
| Quality | Lint、SonarQube、依赖漏洞扫描 | 视规则拦截或警告 |
| Package | 构建镜像/制品并打版本标签 | 终止 |
| Deploy | 部署到目标环境（滚动/蓝绿/金丝雀） | 自动回滚 |
| Verify | 健康检查、冒烟测试 | 告警并回滚 |

原则：前面的阶段越快越好（目标主干构建 10 分钟内）；慢测试拆到夜间流水线，别堵住每次提交。

### 制品与版本

- 制品（Artifact）是构建的唯一产物：jar、Docker 镜像、前端静态包。**一次构建、处处部署**——同一制品从测试环境晋级到生产，禁止在每个环境重新构建。
- 版本标识 = Git commit SHA 或语义版本 + 构建号；镜像标签用 SHA，不要只用 `latest`（无法定位与回滚）。

## 3. 进阶知识

### GitHub Actions 与 Jenkins 对比

| 维度 | GitHub Actions | Jenkins |
|---|---|---|
| 托管 | GitHub 托管 runner，开箱即用 | 自建 Controller/Agent |
| 配置 | 仓库内 `.github/workflows/*.yml` | Jenkinsfile 或页面配置 |
| 生态 | Marketplace actions 丰富 | 插件极多但质量参差 |
| 成本 | 公共仓库免费；私有仓库按分钟计费 | 服务器与运维成本自担 |
| 适用 | 代码本就在 GitHub、想快速落地 | 内网/合规要求、自有基础设施多 |
| 扩展 | 自托管 runner 可接入内网 | 节点、凭据、网络完全可控 |

选型经验：代码在 GitHub 上优先 Actions（ao 的知识库就在 GitHub，零运维）；银行/内网项目（公司项目环境）大多还是 Jenkins + 内网制品库，因为代码与环境出不了内网。

### 部署策略

- **滚动发布**：分批替换实例，简单，但新旧版本短暂共存，要求向后兼容。
- **蓝绿发布**：两套完整环境切换流量，回滚最快，资源成本翻倍。
- **金丝雀**：先放 5% 流量验证指标，再逐步放量；依赖监控与自动回滚（见 [[监控告警]]）。

## 4. 原理剖析

流水线本质是 DAG 任务调度：job 之间用 `needs` 声明依赖，可并行的并行（如多模块测试），关键路径决定总时长。制品仓库（Docker Registry、Nexus、GitHub Packages）是流水线与部署之间的解耦层：构建侧只管推，部署侧只管拉指定版本。

凭据管理是安全核心：密钥走平台的 Secrets 注入环境变量，绝不写进仓库；自托管 runner 视为高权限机器，按最小权限管理。

## 5. 实战应用

### GitHub Actions：Spring Boot 项目流水线

```yaml
name: ci
on:
  push:
    branches: [ main ]
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven
      - name: Build & Test
        run: ./mvnw -B verify
      - name: Build Image
        if: github.ref == 'refs/heads/main'
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

要点：PR 只跑构建+测试（不推镜像）；主干才产出制品；Maven 依赖缓存让构建从几分钟降到几十秒。

### Jenkins 等价骨架（公司内网项目常见）

```groovy
pipeline {
  agent any
  stages {
    stage('Build') { steps { sh './mvnw -B -DskipTests package' } }
    stage('Test')  { steps { sh './mvnw -B verify' } }
    stage('Image') { steps { sh "docker build -t app:${env.GIT_COMMIT} ." } }
    stage('Deploy') {
      steps { sh 'kubectl set image deploy/app app=app:${GIT_COMMIT}' }
    }
  }
  post { failure { echo 'pipeline failed, notify owner' } }
}
```

### 回滚

- 镜像回滚：把部署的镜像标签切回上一个 SHA，`kubectl rollout undo` 或改 compose 的 image 后重新拉起。
- 数据库变更是回滚难点：迁移脚本坚持向前兼容（先加列后删列分两次发布），否则回滚代码也救不回数据结构。

## 6. 疑难杂症

- **本地能过、CI 挂**：九成是环境差异——时区、JDK 小版本、依赖缓存脏了、测试对执行顺序有依赖。复现方法：在 CI 同款镜像里本地跑一遍。
- **测试不稳定（flaky）**：依赖真实时间、随机数、外部网络的测试要隔离或打桩；flaky 测试先 quarantine 并立单追踪，别直接重试掩盖。
- **流水线越来越慢**：拆并行 job、加依赖缓存、把全量测试分层（PR 跑快测，合并后跑全量）。

## 7. 常见误区

- 把部署脚本和构建脚本混在一起，换环境就重构建，破坏"一次构建处处部署"。
- 密钥写进仓库或打印到日志；日志里 `echo $TOKEN` 是经典事故。
- 只做 CI 不做自动部署验证：部署"成功"但服务起不来，要加健康检查步骤才算闭环。

## 8. 性能优化

- 缓存依赖（Maven/npm/pip）与 Docker 层缓存，通常省时最明显。
- 矩阵构建（多 JDK 版本/多模块）并行跑；大单体按模块变更范围只跑受影响模块的测试。
- 自托管 runner 放在离制品库近的网络里，镜像推拉是常见瓶颈。

## 9. 版本差异

- GitHub Actions 的主流 action 版本滚动更新快，`actions/checkout@v4`、`setup-java@v4` 是当前常用组合，写法上已收敛为声明式缓存参数。
- Jenkins LTS 每年多次小版本，插件兼容性要在升级前在测试 Controller 验证，生产上别开自动更新插件。

## 10. 面试与自测

1. CI 与 CD 的区别？持续交付和持续部署差在哪一步？
2. 为什么强调"一次构建、处处部署"？
3. 蓝绿与金丝雀分别适合什么场景？回滚速度与成本如何权衡？
4. 流水线里密钥怎么管？runner 被攻破的影响面是什么？
5. 一条 40 分钟的流水线，你按什么顺序优化？

### 参考答案

1. CI 是代码合入主干后自动完成构建、静态检查、单元测试，让问题在提交后几分钟内暴露；CD 分两档：持续交付是流水线产出可发布制品、自动部署到测试/预发环境，但上线保留人工批准，持续部署则全自动直达生产。两者差在最后一步：持续交付多一次人工批准，持续部署没有。
2. 同一制品从测试环境晋级到生产，禁止在每个环境重新构建。重构建会引入环境差异（依赖版本、编译参数、机器状态），导致测过的东西不等于上线东西，出了问题也无法按制品追溯和回滚；版本标识用 Git commit SHA，不要只用 latest。
3. 蓝绿适合发布风险高、回滚速度优先的场景：两套完整环境切换流量，回滚最快，但资源成本翻倍。金丝雀适合需要灰度观察的场景：先放 5% 流量验证指标，再逐步放量，成本低，但依赖监控与自动回滚。权衡看三点：回滚速度、资源成本、观察能力。
4. 密钥走平台的 Secrets 注入环境变量，绝不写进仓库、不打印到日志（日志里 echo $TOKEN 是经典事故）。自托管 runner 应视为高权限机器：它能触及制品、部署目标和凭据，按最小权限管理；一旦被攻破，影响面就是它能部署、能发版、能读到的全部凭据和环境。
5. 按省时确定性从高到低：先加缓存（Maven/npm 依赖缓存与 Docker 层缓存，通常省时最明显）；再做并行（矩阵构建、多模块并行、只跑受影响模块的测试，流水线本质是 DAG 调度，关键路径决定总时长）；再把测试分层（PR 跑快测、合并后跑全量、慢测试拆到夜间）；最后把自托管 runner 放到离制品库近的网络里，镜像推拉是常见瓶颈。主干构建目标是 10 分钟内。
## 11. 关联知识

- 前置：[[Git-运维与工具]]（分支策略与主干开发）、[[Docker]]（镜像与制品）
- 相关：[[监控告警]]（发布后验证与自动回滚依据）、[[Nginx]]（流量切换实现蓝绿/金丝雀）
- 延伸：GitOps（Argo CD）、制品签名与 SBOM 供应链安全
