---
知识点: Python 打包与环境管理
层级: 基础
适用版本: Python 3.x（工具用法以官方文档为准）
更新日期: 2026-10-30
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
