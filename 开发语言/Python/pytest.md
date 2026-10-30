---
知识点: pytest 实战
层级: 基础
适用版本: Python 3.x
更新日期: 2026-10-31
关联:
  - Python
标签:
  - Python
  - 实践
---

# pytest 实战

## 1. 核心概念

pytest 是 Python 的主流测试框架，设计哲学是"约定优于配置"：文件名 `test_*.py`、函数名 `test_*`、直接用原生 `assert`（失败时自动展示变量值，不用记 `assertEqual` 一家子）。相比标准库 unittest，样板代码少一个量级。

- 小说引擎的抽取函数、RAG 的 chunk 函数，先写 3 个用例再写实现，TDD 在这种"输入输出明确"的场景最划算。
- 测试的真正价值不是覆盖率数字，而是**改代码时的回归安全网**：重构 ChapterAnalysisService 前先跑一遍全绿，改完再跑，红了就是改坏了，定位范围天然缩小。

## 2. 基础知识

```python
# test_extract.py
import pytest
from extractor import parse_entities

def test_parse_normal():
    assert parse_entities("林溯走进源石矿") == ["林溯", "源石矿"]

@pytest.mark.parametrize("text,expected", [
    ("", []),
    ("无实体的纯风景描写", []),
])
def test_parse_edge(text, expected):
    assert parse_entities(text) == expected
```

```bash
pytest -q                    # 跑全部
pytest test_extract.py -k edge -v
pytest --lf                  # 只跑上次失败的
```

- `parametrize` 一行变 N 个用例，边界条件全覆盖；参数组合多时还可叠加多个装饰器做笛卡尔积。
- 断言异常：`with pytest.raises(ValidationError): ...`，还能用 `match=` 校验异常消息的正则。
- 目录约定：测试文件放 `tests/` 并与源码包分开；在项目根放空的 `conftest.py` 能让 pytest 把根目录加入 `sys.path`，解决"本地能跑、CI 里 import 不到包"的经典问题。

## 3. fixture：测试的依赖注入

fixture 是 pytest 的核心机制：用 `@pytest.fixture` 声明"准备好的对象"，测试函数把名字写进参数列表即可注入，类似 Spring 的依赖注入，但作用域和生命周期由 `scope` 控制。

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

@pytest.fixture(scope="session")
def engine():
    # 整个测试会话只建一次：内存 SQLite，起得快
    return create_engine("sqlite://")

@pytest.fixture()
def session(engine):
    # 每个用例独立 session，结束回滚，互不污染
    with Session(engine) as s:
        yield s
        s.rollback()

def test_create_entity(session):
    ...
```

- `scope` 取值：`function`（默认，每用例重建）< `class` < `module` < `session`。重的资源（建表、加载 embedding 模型）往大作用域放，隔离需求往小作用域放。
- fixture 里 `yield` 之前是 setup、之后是 teardown；也可以用 `yield` 把对象交给测试。
- `conftest.py` 里定义的 fixture 对同目录及子目录的测试自动可见，不用 import——共享的 DB、临时目录、mock 客户端都放这里。
- 内置好用的 fixture：`tmp_path`（每用例独立的临时目录，测文件读写/RAG 落盘不用自己清理）、`monkeypatch`（临时改环境变量、属性、函数，用例结束自动恢复）、`capsys`（捕获 print 输出）。

## 4. mock 外部依赖：别真调 Ollama

单测里绝不能真发 LLM 请求：慢、不稳定、结果不可复现。做法是把"调模型"这一层抽成函数/客户端，测试时用 `monkeypatch` 或 `unittest.mock` 换成假实现，返回写死的 JSON：

```python
def test_analysis_pipeline(monkeypatch):
    fake_json = '{"entities": [{"name": "林溯", "type": "人物"}], "events": []}'
    monkeypatch.setattr("analysis.llm_chat", lambda messages: fake_json)
    result = analyze_chapter("林溯推开矿洞的门")
    assert result.entities[0].name == "林溯"
```

- 测的是**你自己的管道**（解析、校验、落库、矛盾检测），不是模型聪不聪明；模型质量另走抽样评测，不混进单测。
- 同理：测 RAG 时 embedding 接口也要 mock 成固定向量，否则向量随模型版本漂移，用例时红时绿。
- 需要断言"调用发生过、参数对不对"时用 `unittest.mock.Mock` / `MagicMock`，`mock.assert_called_once_with(...)`。

## 5. 标记、筛选与组织

- 自定义标记：在 `pytest.ini` / `pyproject.toml` 里注册 `markers`，如 `slow`、`integration`；跑单测时 `pytest -m "not slow"` 跳过慢的集成用例，CI 分开跑。
- `pytest -x` 第一个失败就停，适合本地调试；`--maxfail=3` 折中。
- 跳过与预期失败：`@pytest.mark.skip(reason=...)`、`@pytest.mark.xfail`（已知 bug 先标着，修好后会报 XPASS 提醒你摘标记）。
- 异步代码：`asyncio` 的协程测试函数需要 `pytest-asyncio` 插件（配置 `asyncio_mode = auto` 后直接写 `async def test_...`），或在同步用例里 `asyncio.run(...)` 包一层，不装插件也能测。

## 6. 配置文件

统一放 `pyproject.toml`，避免参数散落在命令行：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
  "slow: 需要真实外部服务的慢速用例",
  "integration: 集成测试",
]
addopts = "-q"
```

覆盖率用 `pytest-cov`：`pytest --cov=novel_engine --cov-report=term-missing`，重点看核心服务层（矛盾检测、状态机）的行覆盖，工具函数不必强求数字。

## 7. 常见误区

- **用例之间有顺序依赖**：A 建的数据 B 接着用——单跑 B 就红。每个用例必须自备数据、独立可跑，靠 fixture 隔离，不靠执行顺序。
- **在测试里写 sleep 等异步/等落盘**：慢且 flaky。等条件轮询要设上限，或直接把时间/IO 抽象出来 mock 掉。
- **断言太松或太严**：只断言"不抛异常"等于没测；断言 LLM 返回的完整字符串又必然脆裂。断言落在结构和关键字段上（类型、数量、必填字段、状态迁移）。
- **把测试当脚本随手改红不管**：红着的用例留久了就没人信 suite 了，要么修要么删要么 xfail 并注明原因。

## 小技巧与冷知识

- 【小技巧】`--ff`（`--failed-first`）比 `--lf` 更适合"修完再全跑"的场景：它先跑上次失败的用例，确认修好了，再按正常顺序跑其余用例；而 `--lf` 只跑失败的用例，修完后想顺手验证全量还得再跑一次。本地改完一个红用例后用 `pytest --ff -q`，一次命令完成"先验修好的、再验没改坏的"。
- 【小技巧】`pytest --durations=10` 跑完后列出最慢的 10 个用例。慢用例往往是偷偷调了真实服务、写了 sleep、或在 fixture 里重复建表的地方；suite 越来越慢时先看这张榜，再决定把谁 mock 掉或把 fixture 作用域放大，比凭感觉优化有效。
- 【小技巧】失败一多就用 `--tb=short`，CI 日志里只想扫失败清单时用 `--tb=line`：默认的长回溯（long）在几十个失败时刷屏，`--tb=line` 每个失败只留一行摘要，先定位"挂了哪些用例"，再对单个用例切回默认回溯细查。回溯风格是展示层面的选择，不影响用例本身的通过与否。
- 【冷知识】`--stepwise`（`-sw`）可以在失败处断点续跑：它在第一个失败时停下，修好代码后再次运行 `pytest --stepwise`，会跳过已经通过的用例、直接从上次失败的用例继续；而 `-x` 每次都要从头跑。调试一串相互关联的失败用例时，stepwise 省掉大量重复等待。
- 【小技巧】断言浮点数别写 `==`，用 `pytest.approx`：`assert 0.1 + 0.2 == pytest.approx(0.3)`，默认相对容差是 1e-6；它还能直接比较列表和字典，逐项按容差判定。RAG 里断言相似度分数、embedding 余弦值这类浮点结果时，approx 能避免"差 1e-16 就红"的假失败。
- 【小技巧】可选依赖的测试文件顶部写 `pytest.importorskip("xxx")`：该库没装时整文件跳过（skip）而不是报错，CI 矩阵里"有的环境装了、有的没装"的用例就不用到处写 try/except。它只在 import 失败时跳过，库存在但版本不对时不会帮你兜底。
- 【冷知识】pytest 的断言重写只发生在测试模块：它改写 `test_*.py` 里 `assert` 的字节码，失败时才能展开变量值；被测源码里的 `assert` 失败时只有一句普通的 AssertionError，没有变量展开。所以想看清"哪边的值不对"，断言要写在测试函数里，别指望源码里的 assert 也给出漂亮的 diff。
- 【小技巧】`capsys` 只能抓 print，想断言日志走 `caplog`：`caplog` fixture 捕获 logging 输出，`caplog.set_level(logging.DEBUG)` 可以临时把级别调低，`caplog.text` 里按字符串断言"某分支有没有打出关键日志"。测"矛盾检测命中规则时是否记了审计日志"这类需求，用 caplog 比在 capsys 里找 print 靠谱。
- 【小技巧】参数化 fixture 让一个用例跑多套环境：`@pytest.fixture(params=["memory", "file"])` 配合 `request.param` 取值，每个引用它的用例会按每个参数各跑一遍。测"同一段落库逻辑在内存 SQLite 和文件 SQLite 下都对"时，比复制两份用例干净，失败时用例名里自带参数值，定位也方便。
- 【小技巧】给 parametrize 的用例起可读名字：`@pytest.mark.parametrize("text,expected", [...], ids=["空串", "纯风景"])`，失败时 `-v` 列表里一眼看出是哪组参数挂了，而不是 `test_parse_edge[text0-expected0]` 这种默认编号。参数组合一多，ids 就是失败清单的可读性。
- 【冷知识】`--lf` 依赖项目根目录下的 `.pytest_cache`：上次失败记录存在 `.pytest_cache/v/cache/lastfailed` 里；删掉它、换目录跑、或 CI 每次全新 checkout，`--lf` 就认不出失败历史。所以 `--lf` 是本地开发的工具，在全新拉取的 CI 环境里用它等于跑全部。
- 【小技巧】`-o` 可以临时覆盖配置而不改文件：`pytest -o addopts=""` 一次性清空 pyproject.toml 里的 addopts；`pytest -o log_cli=true` 临时打开实时日志输出。想试一个配置效果、又不想把实验性改动留在配置文件里时，用 `-o` 最干净。
- 【小技巧】临时改环境变量用 `monkeypatch.setenv` / `monkeypatch.delenv`，别直接改 `os.environ`：monkeypatch 在用例结束时自动恢复原值，不会污染后面的用例；`monkeypatch.delenv("API_KEY", raising=False)` 还能覆盖"缺变量"这个分支。原文的 monkeypatch 换函数是同一机制在环境变量上的用法。
- 【冷知识】`@pytest.mark.xfail(strict=True)` 会把"意外通过"变成失败：默认的 xfail 用例意外通过只报 XPASS、不影响整体通过；strict=True 时 XPASS 直接算失败，逼着修好 bug 后立刻摘掉标记。原文说的"修好后会报 XPASS 提醒你摘标记"就是这个机制，严格模式下提醒变成强制，适合"已知 bug 必须有人跟进"的团队。

## 面试与自测

1. fixture 的 scope 有哪几档？重的资源和隔离需求分别往哪档放？
2. 单测里为什么不能真调 Ollama 或 embedding 接口？正确的做法是什么？
3. 用例之间有顺序依赖（A 建的数据 B 接着用）为什么不行？怎么修？

### 参考答案

1. scope 取值：function（默认，每用例重建）< class < module < session。重的资源（建表、加载 embedding 模型）往大作用域放，隔离需求往小作用域放；fixture 里 yield 之前是 setup、之后是 teardown。
2. 因为慢、不稳定、结果不可复现；向量还会随模型版本漂移，用例时红时绿。做法是把“调模型”这一层抽成函数/客户端，测试时用 monkeypatch 或 unittest.mock 换成假实现，返回写死的 JSON（embedding 接口 mock 成固定向量）；测的是你自己的管道（解析、校验、落库、矛盾检测），不是模型聪不聪明。需要断言“调用发生过、参数对不对”时用 Mock / MagicMock 的 assert_called_once_with(...)。
3. 不行，因为单跑 B 就红——每个用例必须自备数据、独立可跑，靠 fixture 隔离，不靠执行顺序。

## 8. 关联知识

- [pydantic](开发语言/Python/pydantic.md)：分析管道的校验层是重点被测对象，非法 JSON 的用例都冲着它写
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：测试的断言对象尽量用 dataclass 而非裸 dict，比较和报错都更清晰
- [asyncio](开发语言/Python/asyncio.md)：批量抽取的并发限流逻辑（Semaphore 上限）值得单独用例验证

## 9. 一句话总结

pytest 的投入产出比在"输入输出明确的纯函数 + 一层管道"上最高：先给抽取、切分、校验、矛盾检测这些核心逻辑织网，再放心重构。
