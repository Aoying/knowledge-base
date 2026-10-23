---
知识点: dataclass 与 typing
层级: 基础
适用版本: Python 3.10+
更新日期: 2026-10-23
关联:
  - Python
  - pydantic
  - asyncio
标签:
  - Python
  - 实践
---

# dataclass 与 typing

## 1. 核心概念

dataclass 是标准库的类生成器：给类加上 `@dataclass`，模块会按字段声明自动生成 `__init__`、`__repr__`、`__eq__` 等样板方法。typing 则是给变量、参数、返回值标注类型的语法（`list[str]`、`Optional[X]`、`dict[str, Any]` 等）。两者配合的目的只有一个：**把"靠约定和注释维持的数据结构"变成"IDE 和类型检查器能验证的结构"**。

- 小说引擎的 Entity/Relation 传来传去全用 dict 的话，字段名拼错只有运行时才知道；dataclass + 类型注解让 IDE 直接标红。
- 类型注解在运行时默认不强制（Python 不会因为注解报错），它的价值在静态检查（IDE、mypy）和代码可读性；需要运行时校验时上 pydantic（见关联知识），轻量结构用 dataclass，零依赖。

## 2. 基础知识

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Chapter:
    id: int
    title: str
    summary: str = ""
    entities: list[str] = field(default_factory=list)  # 可变默认值必须用它

def get_chapter(cid: int) -> Optional[Chapter]: ...
```

要点：

- **可变默认值必须用 `field(default_factory=...)`**：直接写 `entities: list = []` 会报 `ValueError: mutable default`。原因是类属性级别的默认值会被所有实例共享——这与 Java 里静态字段被共享是同一个坑，Python 干脆在定义期就拦下来。
- **字段顺序**：无默认值字段必须排在有默认值字段之前，和函数参数规则一致。
- **`@dataclass` 常用参数**：
  - `frozen=True`：实例不可改属性（赋值抛 `FrozenInstanceError`），且自动可哈希，可当 dict 的 key 或放进 set——适合表示"值对象"，如一条不可变的剧情事实 StoryFact。
  - `eq=True`（默认）：按字段逐个比较生成 `__eq__`。
  - `slots=True`（3.10+）：用 `__slots__` 存字段，省内存、属性访问更快、且拼错属性名直接报错；大量小对象（如全量 EntityState 快照）时值得开。
  - `kw_only=True`（3.10+）：强制构造时用关键字传参，字段多时防传错位。

```python
@dataclass(frozen=True, slots=True)
class StoryFact:
    subject: str          # 林溯
    predicate: str        # 不知道
    obj: str              # 父母已死亡
    since_chapter: int    # 从第几章起成立
```

## 3. typing 常用工具

- `Optional[X]` 等价于 `X | None`（3.10+ 推荐写 `X | None`）。
- `list[str]`、`dict[str, int]`：3.9+ 内置泛型可直接用，不必再从 typing 导入 `List/Dict`。
- `Literal["planted", "developed", "paid_off"]`：把取值限定为字面量集合，伏笔台账 Hook 的状态字段用它，传错字符串 IDE 直接标红，比枚举更轻、比裸 str 安全。
- `TypedDict`：给"形状固定的 dict"标注类型，适合描述外部 JSON 的原始形态（LLM 返回的裸 JSON 先声明成 TypedDict，再转 dataclass/pydantic）。
- `Protocol`：结构化类型，"长得像就行"（鸭子类型的静态版）。定义工具接口时比继承 ABC 更松耦合，Agent 的 Tool 抽象常用：

```python
from typing import Protocol

class Tool(Protocol):
    name: str
    def run(self, args: dict) -> str: ...   # 任何有 name 和 run 的类都算 Tool
```

- `TypeAlias` / 3.12 的 `type` 语句：给复杂类型起别名，如 `EntityId = int`，让签名自解释。

## 4. 进阶：dataclass 与其他方案的取舍

| 方案 | 运行时校验 | 依赖 | 适用 |
|---|---|---|---|
| 裸 dict | 无 | 无 | 一次性脚本、透传 |
| dataclass | 无 | 标准库 | 进程内部传递的结构（主力） |
| TypedDict | 无 | 标准库 | 描述外部 dict 的形状 |
| pydantic | 有 | 第三方 | 外部输入边界：LLM 输出、API 入参 |
| namedtuple | 无 | 标准库 | 极轻量只读记录，现多被 dataclass 取代 |
| attrs | 部分 | 第三方 | dataclass 的"完全体"，新项目一般不需要 |

经验分工：**边界用 pydantic 校验，内部用 dataclass 传递**。LLM 吐的 JSON 经 pydantic 验证后转成 dataclass 对象进入业务层，业务层里就不必再到处防空指针式地 `.get()`。

## 5. 实战应用

从 dict 安全地构造 dataclass：字段对不上时直接 `Chapter(**d)` 会因多余键报错。小说引擎里 JSON 来源杂，常用一个宽容的构造助手：

```python
import dataclasses

def from_dict(cls, data: dict):
    names = {f.name for f in dataclasses.fields(cls)}
    return cls(**{k: v for k, v in data.items() if k in names})

ch = from_dict(Chapter, {"id": 3, "title": "重逢", "junk": 1})  # 多余字段丢弃
```

序列化反方向：`dataclasses.asdict(ch)` 递归转回 dict（落 SQLite 的 JSON 列、写日志都方便）；只取字段名用 `dataclasses.fields(cls)`；复制并改一两个字段用 `dataclasses.replace(ch, summary="新摘要")`——对 frozen 类尤其有用，不可变对象靠 replace 产生新版本，正好对应 EntityState 的"状态变更生成新记录"模型。

与 SQLAlchemy 的关系：引擎里持久化实体已经有 ORM 模型，不要再拿 dataclass 重复建模同一张表；dataclass 适合的是**非持久化的中间结构**：ChapterContextService 聚合出的"世界状态上下文"、ChapterAnalysisService 的分析结果 DTO 这类算完即弃或整体序列化的数据。

## 6. 常见误区

- **`frozen=True` 不是深度不可变**：字段里装的 list/dict 本身仍可改，冻结的只是属性绑定。
- **注解 ≠ 校验**：`def f(x: int)` 传字符串照样跑；想要运行时拦截只有 pydantic 或手写检查。反过来，注解写错类型（如实际返回 None 却标 `-> Chapter`）会让 IDE 的提示变成误导，比不写更糟。
- **继承 dataclass 时的字段顺序坑**：父类带默认值的字段会把子类字段全部"传染"成必须有默认值，设计继承体系时先想清楚，或用 `kw_only=True` 化解。
- **`field(default_factory=list)` 写成 `default=list()`**：后者是定义时求值一次的共享实例，就是要避开的那个坑本身。

## 7. 关联知识

- [pydantic](开发语言/Python/pydantic.md)：运行时校验的一侧，与本篇是边界/内部分工关系
- [asyncio](开发语言/Python/asyncio.md)：异步函数的签名同样靠类型注解保持可读
- [装饰器](开发语言/Python/装饰器.md)：`@dataclass` 本身就是类装饰器

## 8. 一句话总结

dict 靠记忆，dataclass + typing 靠工具检查；项目一大，先把核心数据结构从 dict 升级成带注解的 dataclass，收益立竿见影。
