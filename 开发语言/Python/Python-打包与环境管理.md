---
知识点: Python 打包与环境管理
层级: 基础
适用版本: Python 3.x（工具用法以官方文档为准）
更新日期: 2026-10-31
关联:
  - Python
  - pytest
标签:
  - Python
  - 实践
  - 速查
---

# Python 打包与环境管理

## 1. 核心概念

Java 有 Maven 管依赖与构建、JDK 版本自成体系；Python 的对应面是三件事分开的：**解释器版本、虚拟环境、依赖清单**。环境出问题十有八九是这三层混在了一起：全局 Python 装了一堆包、项目之间互相污染、换台机器装不回来。管理目标就一条：**每个项目的依赖可声明、可重建、可隔离**。

小说引擎的现行口径：项目独立 venv（`townChat/.venv`），依赖以清单文件为准，全局环境不装业务包。本文把这套口径写清，工具细节（命令参数、pyproject 字段）以各工具官方文档为准，不在此写死版本结论。

## 2. 虚拟环境 venv

venv 是官方内置的隔离机制：一个目录里放一份解释器链接 + 独立的 `site-packages`，激活后 `python` / `pip` 指向这份隔离环境。

```bat
:: Windows（引擎本机口径）
python -m venv .venv
.venv\Scripts\activate
python -m pip install -r requirements.txt
deactivate
```

- **建环境用 `python -m venv`**，别用裸 `venv` 命令或第三方工具混着来；选哪个解释器建，环境里就是哪个版本——建之前先 `python --version` 确认自己用的是哪个 Python（本机多版本共存时这是头号事故源）。
- 用 `python -m pip` 而不是裸 `pip`：保证 pip 和当前解释器是同一家，裸 `pip` 指向另一个 Python 是经典环境串味。
- `.venv` 不进版本库（进 `.gitignore`），它可由清单随时重建；进库的是清单，不是环境本身。
- IDE（PyCharm/VS Code）要手动把解释器指到项目 `.venv`，否则编辑器里不报红、命令行跑起来缺包，两边各说各话。

## 3. pip 与 requirements

```bash
python -m pip install sqlalchemy pydantic
python -m pip freeze > requirements.txt     # 导出当前环境的精确版本
python -m pip install -r requirements.txt    # 按清单重建
```

- `pip freeze` 导出的是**全量精确锁定**（含间接依赖），适合"把本机能跑的状态原样搬走"；手写 requirements 只列直接依赖并给版本范围，适合长期维护的项目声明。两种口径别混用一份文件：引擎这种个人长期项目，建议直接依赖手写维护 + 定期 freeze 出一份锁定备份。
- 版本约束写法：`==` 锁死、`>=` 下限、`~=` 兼容版本（`~=2.0` 表示 `>=2.0, <3.0`）。关键依赖（SQLAlchemy、pydantic 这类大版本间 API 有差异的）建议锁大版本范围，别裸名不带约束——某天重建环境装到新大版本，代码在无人值守时悄悄失效。
- 装不上先查三件事：是不是在 venv 里、pip 对应的解释器版本对不对、是不是拿了没激活的环境在装。

## 4. pyproject.toml

`pyproject.toml` 是现在的统一项目元数据文件（构建、依赖、工具配置都往里收），逐步取代 `setup.py` + 多个散配置文件：

```toml
[build-system]
requires = ["setuptools>=61"]
build-backend = "setuptools.build_meta"

[project]
name = "novel-tool"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "sqlalchemy>=2.0,<3.0",
    "pydantic>=2.0,<3.0",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

- 对引擎这种**应用型项目**（不发布到 PyPI），pyproject 的主要价值是把依赖声明和各工具配置（pytest、ruff 等）收进一个文件、给编辑器和后续工具一个统一入口；不必为了"正规"硬套打包发布流程。
- `dependencies` 是给打包工具看的声明，与 requirements.txt 的分工：pyproject 写"项目需要什么"，锁定文件写"这次到底装了哪一版"。用 uv/pip-tools 这类工具时由工具从 pyproject 生成锁定，别手工维护两份还让它们打架。

## 5. uv 简介

uv 是近年来主流的 Rust 编写的 Python 包与项目管理工具，一个二进制覆盖建环境、装依赖、锁定、跑脚本，还能管理 Python 解释器版本本身；速度是它的主要卖点。对引擎的定位建议：

- **值得试，不必急着迁移**：新项目可以直接用 uv 起（`uv init` / `uv add sqlalchemy` / `uv run`），工作流比 pip + venv 更顺。
- 已稳定的 `townChat/.venv` + requirements 口径不要为了换工具推翻——迁移的收益是速度和锁定文件，成本是全队（虽然就一个人）习惯与脚本改动。真要迁，先在一个小脚本项目上验证 `uv run` 与现有 IDE 配置是否顺手，再动引擎。
- uv 的具体命令与配置以官方文档为准，工具迭代快，本文不写死命令清单。

## 6. 常见误区

- **往全局 Python 装业务依赖**：短期省事，长期串味；判断标准是——这个包换个项目还用吗，不用就只配装进项目 venv。
- **只装不记**：手敲 `pip install` 装上了却没写进任何清单，环境即成一次性用品。养成"装完立刻更新清单"的习惯，或用会自动改清单的工具（如 `uv add`）。
- **拿 freeze 的产物当项目声明长期维护**：里面混着一堆间接依赖与本机偶然装的包，别人（或未来的自己）看不出哪些是真需要的；定期回看、给直接依赖补注释分组。
- **多个项目共用一个 venv**：省磁盘，赔隔离；升级一个项目的依赖打坏另一个，是迟早的事。
- **`.venv` 整个拷给另一台机器**：路径与解释器链接写死在里面，拷过去多半是坏的；搬运永远走"清单 + 重建"。

## 小技巧与冷知识

【小技巧】装包之前先空跑一遍：`pip install --dry-run -r requirements.txt` 只算依赖、不真装，能提前看到会装哪些包、哪些版本冲突；装完再跑 `pip check`，它会验已装包之间的依赖是否自洽。重建环境后先 `pip check` 再跑测试，依赖冲突早发现比测试跑一半报 ImportError 省事得多。

【小技巧】对来源有要求时用哈希锁定：`pip install --require-hashes -r requirements.txt` 要求清单里每行都带 `--hash=sha256:...`，下载的包哈希对不上就拒绝安装，防中间人篡改。哈希可以用 `pip-compile --generate-hashes`（pip-tools）自动生成；内网可信源没必要上这套，公网拉包、或要给别人交付可复现环境时再上。

【小技巧】换镜像源写进配置，别每次命令后贴 `-i`：`pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple`，或设环境变量 `PIP_INDEX_URL`，本地和 CI 用同一份配置。命令行临时加 `-i` 只适合一次性的试装，长期靠它等于把源地址散在每个人的 shell 历史里，换源时没人记得全。

【小技巧】系统 Python 报 `externally-managed-environment` 时别加 `--break-system-packages` 绕过去：Debian 12、Ubuntu 23.04 之后的系统 Python 归 apt 管，pip 直接往里装包会和系统包打架，PEP 668 才加了这道锁。正确的反应是建个 venv 再装，`--break-system-packages` 是"我知道风险"的开关，不是"修好了"的开关。

【小技巧】新建 venv 时顺手把 pip 升到最新：`python -m venv --upgrade-deps .venv` 会在建环境时就把 pip 和 setuptools 升到最新版；旧 pip 可能认不出新格式的 wheel、或解析新版依赖元数据失败。装包报错先看 pip 版本，比先怀疑包本身更划算，`python -m pip install -U pip` 是环境出问题时的第一步排查。

【小技巧】判断当前在哪个环境看 `sys.executable`，别只看提示符：`python -c "import sys; print(sys.executable)"` 直接告诉你现在用的解释器路径，`echo $VIRTUAL_ENV` 看激活的是哪个 venv。Shell 提示符前的 `(.venv)` 可以被 `--prompt` 自定义、也可能被 IDE 隐藏，提示符说在环境里、解释器路径说不在时，以解释器路径为准。

【小技巧】requirements 文件可以拆分：基础依赖放 `requirements.txt`，开发依赖放 `requirements-dev.txt`，开头用 `-r requirements.txt` 包含基础；再用 `-c constraints.txt` 统一钉间接依赖的版本上限，`-c` 只约束版本、不装新包。拆分后"生产装什么、开发多装什么"一目了然，比一个文件里混着 dev 包、还靠注释分组干净得多。

【小技巧】本地开发自己的包用可编辑安装：`pip install -e .` 把项目以"指向源码目录"的方式装进环境，改代码不用重新安装，import 的就是正在写的源码。配合 pyproject 里 `[project] name` 的声明，测试和脚本都能直接 import 项目包；不发布到 PyPI 的应用型项目同样能用这招，不必等"打包发布"才用。

【小技巧】开发依赖用 extras 分组，别塞进主 dependencies：pyproject 里 `[project.optional-dependencies]` 写 `dev = ["pytest", "ruff"]`，装时 `pip install ".[dev]"` 一次装齐。生产环境只装主依赖、开发环境再加 `[dev]`，同一份声明、两套安装组合；把 pytest 写进主依赖等于让生产环境也背上测试工具。

【小技巧】锁文件要进版本库，`.venv` 不进：`requirements.txt` 或 `uv.lock` 提交上去，别人 clone 后 `pip install -r requirements.txt` 或 `uv sync` 就能一键重建出同样的环境。锁文件不提交，清单就只是"建议"，重建时装到哪个版本全看当天 PyPI 上有什么；进库的是"这次到底装了哪一版"的证据，不是环境目录本身。

【小技巧】`uv sync` 是精确同步，会删多余的包：它按 `uv.lock` 把环境调成和锁文件完全一致，锁文件里没有的包会被删掉；只想追加、不想删包时用 `uv pip install`。刚从 pip 切到 uv 的人最容易在这里翻车——`uv sync` 之后发现之前手装的包不见了，那不是 bug，是"同步"这个词的本意。

【小技巧】解释器版本也要钉，别只钉包版本：项目根目录放一个 `.python-version` 文件写 `3.12`，uv 和 pyenv 会自动按它选用解释器建环境。requirements 只管包的版本，Python 小版本一换（比如 3.11 升 3.12），某些包的 wheel、语法支持都会跟着变；包和解释器两个版本都钉死，重建才真的可复现。

【小技巧】装包失败先怀疑缓存：`pip cache purge` 清掉本地 wheel 缓存，或加 `--no-cache-dir` 不用缓存重装一次。缓存里的 wheel 损坏、或源换了但缓存还留着旧包时，报错信息往往和真实原因对不上；清缓存是"装不上"排查里成本最低的一步，比换源、升级 pip 都便宜。

【小技巧】升级依赖一次只升一个，升完跑测试：`pip list --outdated` 先看哪些包落后了，再一个一个 `pip install -U 包名` 升级，每升一个跑一遍测试。`pip install -U` 后面不带包名会把所有包全升上去，某个大版本一变、测试全红，根本定位不到是谁干的；升级是改动，改动就要一次只改一处。

【冷知识】venv 是 PEP 405，Python 3.3 才进标准库：在此之前隔离环境靠第三方 virtualenv（2007 年就有了）；pip 也是 PEP 453（Python 3.4）靠 ensurepip 才随解释器自带。也就是说 Python 的"官方环境管理"满打满算只有十多年历史，Java 的 Maven 2004 年就发布了，Python 的包管理是后补的课。

【冷知识】pyproject.toml 是两步定下来的：PEP 518（2016 年）只定了"用 pyproject.toml 这个文件放构建配置"，项目元数据（名字、版本、依赖）是 PEP 621（2020 年）才放进 `[project]` 表的。在此之前一个项目要散着 setup.py、setup.cfg、requirements.txt、MANIFEST.in 四五个文件，pyproject 是把它们收拢成一个文件的第二次尝试。

【冷知识】uv 是 2024 年 2 月发布的，由 Astral 公司（ruff 的作者 Charlie Marsh 创办）用 Rust 编写：它一个二进制覆盖了建环境、装依赖、锁定、跑脚本，还能管理解释器版本本身。uv 出现之前，Python 生态是 pip、venv、pip-tools、virtualenv、pyenv 各管一摊，uv 的卖点之一就是把这几件工具收进一个命令。

【冷知识】pip 的名字是个递归缩写："pip installs packages"，和 GNU 的"GNU's Not Unix"一个套路。wheel 格式则是 PEP 427（2012 年）才定下来的，在此之前分发只有源码包（sdist，tar.gz），装包时要在本机现场编译 C 扩展，没编译器就装不上；wheel 让"下载即用"成为默认，编译只在打包方做一次。

【冷知识】requirements.txt 这个文件名没有任何标准规定：它只是社区约定俗成的名字，pip 本身只认 `-r` 后面给的文件名，叫它 deps.txt、prod.txt 也一样能用。真正被标准化的反而是 pyproject.toml（PEP 518/621），requirements.txt 至今仍是"大家都这么叫"的约定，不是规范。

## 面试与自测

1. 建 venv 之前为什么要先确认 python --version？装包为什么要用 python -m pip 而不用裸 pip？
2. pip freeze 的产物和手写 requirements 各适合什么场景？为什么不能混用一份文件？
3. .venv 为什么不能整个拷给另一台机器？正确的搬运方式是什么？

### 参考答案

1. 因为选哪个解释器建，环境里就是哪个版本；本机多版本共存时，建之前不确认自己用的是哪个 Python 是头号事故源。用 python -m pip 而不是裸 pip，是为了保证 pip 和当前解释器是同一家——裸 pip 指向另一个 Python 是经典环境串味。
2. pip freeze 导出的是全量精确锁定（含间接依赖），适合“把本机能跑的状态原样搬走”；手写 requirements 只列直接依赖并给版本范围，适合长期维护的项目声明。两种口径别混用一份文件；引擎这种个人长期项目，建议直接依赖手写维护 + 定期 freeze 出一份锁定备份。
3. 因为 .venv 里路径与解释器链接是写死的，拷过去多半是坏的。搬运永远走“清单 + 重建”：.venv 不进版本库（进 .gitignore），进库的是清单，不是环境本身。

## 7. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [pytest 实战](开发语言/Python/pytest.md)：测试环境同样跑在这套 venv 与清单之上
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)、[pydantic 数据校验](开发语言/Python/pydantic.md)：清单里最该锁大版本的两个依赖

## 8. 一句话总结

环境管理就三句话：项目独占一个 venv、依赖只认清单文件、换机器永远重建不拷贝；uv 让这套流程更快，但纪律比工具重要。
