---
知识点: dataclass 与 typing
层级: 基础
适用版本: Python 3.10+
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】给不需要参与比较或打印的字段加 `field(compare=False, repr=False)`。比如缓存用的大段文本、最后同步时间这类字段，排除了之后 `==` 只比较真正决定身份的字段，`repr()` 也不会被几百字的内容淹没，日志一下子就干净了。反过来想：默认所有字段都参与 `__eq__`，一个本不该影响相等的缓存字段被改动，就会让两个“同一条事实”的对象突然不相等。

【小技巧】派生字段用 `field(init=False)` 加 `__post_init__` 来算，不要让调用方传。比如由 `title` 算出的 `slug`、由起止章节算出的 `span`，构造时只传原始字段，`__post_init__` 里统一算出来。这样构造签名保持干净，也避免调用方传了派生值却和原始字段对不上的自相矛盾。

【冷知识】类级别常量必须标 `ClassVar`，否则它会变成一个字段。写 `MAX_RETRY: int = 3` 时，dataclass 会把它当成带默认值的字段收进 `__init__`；写成 `MAX_RETRY: ClassVar[int] = 3` 才会被排除在 `dataclasses.fields()` 之外。这个坑很隐蔽：类定义看着没问题，直到有人构造时发现多了一个本不该传的参数，或者 `asdict()` 里多出一项。

【小技巧】只在构造时用、构造完不保存的参数用 `InitVar`。比如传一个原始字符串 `raw` 进来，在 `__post_init__` 里解析成结构化字段，解析完 `raw` 本身不留在实例上。它不会出现在 `fields()`、`asdict()` 和 `repr()` 里，实例上干干净净。适合“输入形态和存储形态不一样”的场景，比如 LLM 返回的原始文本先进来、解析后的字段留下来。

【冷知识】`dataclasses.asdict()` 返回的是深拷贝，不是原对象的浅视图。它内部对非 dataclass 的值用了 `copy.deepcopy`，所以改返回的 dict 里嵌套的 list，不会反过来污染原实例。利用这一点可以放心地把 `asdict()` 的结果拿去改、拿去序列化；但也意味着对超大对象频繁调用 `asdict()` 会有拷贝开销，别把它当成零成本的“只读快照”。

【冷知识】`dataclasses.replace()` 改不了 `init=False` 的字段，调用会直接报 `ValueError`。因为 `replace()` 本质是“用新参数重新走一遍 `__init__`”，而 `init=False` 的字段根本不在构造参数里。想改派生字段时要么重新构造整个对象，让 `__post_init__` 重算；要么把该字段改回 `init=True`。这个限制对 frozen 类尤其值得注意：`replace()` 本是 frozen 对象产生新版本的主力手段，派生字段却走不通这条路。

【冷知识】`frozen=True` 不保证对象一定可哈希，字段里只要有一个不可哈希的类型（如 list、dict），`hash()` 就会报 `TypeError`。冻结只保证“属性绑定不能改”，不保证“字段值本身可哈希”。所以把 frozen 实例当 dict 的 key 或放进 set 之前，要确认所有字段都是可哈希类型；含 list 的字段可以先转成 tuple，或者干脆不要指望它可哈希。

【小技巧】`Annotated` 可以在不改变类型的前提下给字段挂元数据，类型检查器会直接忽略附加信息。比如 `name: Annotated[str, "max_len=20"]`，静态检查时它就是 `str`，运行时用 `get_type_hints(Cls, include_extras=True)` 又能把 `"max_len=20"` 读出来。适合给字段贴“长度上限、单位、来源”这类备注，让校验逻辑或文档生成去读它，而不是另建一套并行的元数据表。

【小技巧】做运行时类型内省（比如按字段类型做通用 `from_dict` 转换）时，用 `get_type_hints()` 而不要直接读 `fields()[i].type`。如果文件头写了 `from __future__ import annotations`，字段的 `.type` 只是字符串 `'int'`，拿它做 `isinstance` 判断会直接失败；而 `get_type_hints()` 会把字符串注解解析回真正的类型对象。这条在写通用序列化、校验工具时最容易踩到。

【小技巧】本篇目标版本是 Python 3.10+，写泛型 dataclass 时优先用 `Generic[T]` 而不是 3.12 才引入的 `class Stack[T]:` 语法。比如 `class Box(Generic[T]): item: T` 在 3.10、3.11、3.12 上都能跑，而方括号新语法只在 3.12+ 有效。团队或部署环境版本不统一时，贪图新语法会让 3.10/3.11 的环境在 import 阶段就直接报错。

【小技巧】循环引用的类型只在注解里出现时，把 import 藏进 `if TYPE_CHECKING:`，运行时不导入，静态检查照常通过。比如 A 模块的 dataclass 字段注解引用 B 模块的类，而 B 又引用 A，顶层互相 import 会报循环导入；把其中一边的 import 移到 `TYPE_CHECKING` 分支下、注解写成字符串，dataclass 本身不求值注解，构造和运行都不受影响，IDE 和 mypy 依然能解析类型。

## 面试与自测

1. `entities: list = []` 为什么会报 `ValueError: mutable default`？正确写法是什么？
2. `frozen=True` 是不是意味着对象里的 list、dict 也不能改了？它还带来什么附带好处？
3. 按本篇的经验分工，LLM 吐的 JSON 应该先过谁、再转成谁进入业务层？这样业务层省掉了什么麻烦？

### 参考答案

1. 因为类属性级别的默认值会被所有实例共享，Python 干脆在定义期就拦下来，直接报 `ValueError: mutable default`。正确写法是用 `field(default_factory=list)`，每个实例各拿一份新的列表。
2. 不是深度不可变：冻结的只是属性绑定，字段里装的 list、dict 本身仍可改。附带好处是实例自动可哈希，可以当 dict 的 key 或放进 set，适合表示“值对象”，如一条不可变的剧情事实 StoryFact。
3. 先过 pydantic 验证，再转成 dataclass 对象进入业务层。分工是边界用 pydantic 校验、内部用 dataclass 传递：dataclass + 类型注解让 IDE 直接标红拼错的字段名，业务层里就不必再到处防空指针式地 `.get()` 了。

## 7. 关联知识

- [pydantic](开发语言/Python/pydantic.md)：运行时校验的一侧，与本篇是边界/内部分工关系
- [asyncio](开发语言/Python/asyncio.md)：异步函数的签名同样靠类型注解保持可读
- [装饰器](开发语言/Python/装饰器.md)：`@dataclass` 本身就是类装饰器

## 8. 一句话总结

dict 靠记忆，dataclass + typing 靠工具检查；项目一大，先把核心数据结构从 dict 升级成带注解的 dataclass，收益立竿见影。
