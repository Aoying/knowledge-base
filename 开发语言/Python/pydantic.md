---
知识点: pydantic 数据校验
层级: 基础
适用版本: pydantic v2
更新日期: 2026-10-31
关联:
  - Python
  - Agent
  - asyncio
标签:
  - Python
  - 实践
---

# pydantic 数据校验

## 1. 核心概念

pydantic 是基于类型注解的**运行时**数据校验库：定义一个继承 `BaseModel` 的类，传入数据时它按注解做类型转换与校验，不合格直接抛 `ValidationError` 并给出结构化的错误清单（哪个字段、什么错、输入是什么）。这补上了 typing 的短板——注解只给 IDE 看，pydantic 真的在运行时拦。

定位：**系统边界的守门员**。一切不可信输入——LLM 输出的 JSON、HTTP 入参、配置文件、外部 API 返回——进系统前先过一遍 pydantic；进了系统内部之后，换 dataclass 传递（见关联知识）。

## 2. 基础知识

```python
from pydantic import BaseModel, Field, ValidationError

class EntityChange(BaseModel):
    name: str = Field(min_length=1)
    type: str = Field(pattern="^(人物|地点|物品|组织)$")
    change: str
    chapter: int = Field(ge=1)

raw = {"name": "林溯", "type": "人物", "change": "得知身世", "chapter": 12}
try:
    ec = EntityChange(**raw)   # LLM 吐的 JSON 先过这一层
except ValidationError as e:
    print("打回重抽:", e.errors()[0])
```

- 小说引擎 ChapterAnalysisService 的 JSON 落库前必须过 pydantic，LLM 偶尔吐 `chapter: "第十二章"` 这种，直接进 SQLite 就是脏数据。校验失败的正确姿势是把 `e.errors()` 拼回提示词让模型重抽（retry with error feedback），而不是吞掉或硬存。
- v2 用 `model_validate` / `model_validate_json`，别再用 v1 的 `parse_obj` / `parse_raw`（v1 已停止维护）。
- `Field` 约束：数值 `ge/gt/le/lt`、字符串 `min_length/max_length/pattern`、集合 `min_length` 等；`Field(default=...)`、`Field(default_factory=...)` 设默认值，规则与 dataclass 一致（可变默认用工厂）。

## 3. 类型转换与严格模式

pydantic 默认是"宽容模式"（lax）：能转就转——字符串 `"12"` 转成 int 12、"true" 转成 bool。这在对接 LLM 时其实是优点（模型常把数字带引号），但要心里有数：

- `chapter: "第十二章"` 转不动 → 报错，正是我们要拦的；
- `chapter: "12"` 会被静默转成 12 → 不报错，想拦这种就开严格模式：字段级 `Field(strict=True)` 或模型级 `model_config = ConfigDict(strict=True)`。
- 枚举字段优先用 `Enum`/`Literal` 而非正则 `pattern`：`type: Literal["人物", "地点", "物品", "组织"]`，报错信息直接列出合法取值，拼回重抽提示时模型更容易改对。

```python
from typing import Literal
from pydantic import BaseModel, ConfigDict

class RelationChange(BaseModel):
    model_config = ConfigDict(extra="forbid")   # 多余字段直接报错，防止模型自创字段
    source: str
    target: str
    kind: Literal["亲属", "同盟", "敌对", "师徒", "恋人"]
    delta: int  # 关系强度变化
```

`extra` 三档：`"ignore"`（默认，丢弃多余字段）、`"allow"`（保留）、`"forbid"`（报错）。LLM 输出场景建议 `forbid`：模型自创的字段往往意味着它没按 schema 理解任务，早暴露早重抽。

## 4. 进阶用法

- **嵌套模型与列表**：分析结果是一整个结构（entities / relations / events / facts），定义顶层 `ChapterAnalysis` 内嵌子模型列表，一次 `model_validate` 全量校验，错误定位精确到 `entities.2.chapter` 这样的路径。
- **字段/模型校验器**：`@field_validator` 做单字段后处理（如把全角数字转半角、strip 空格）；`@model_validator(mode="after")` 做跨字段校验（如"死亡事件的人物不能同时出现在存活名单"这类矛盾的前置拦截，完整的矛盾检测仍归 ContradictionService）。
- **别名**：`Field(alias="entityName")` 对接驼峰命名的外部 JSON；`model_dump(by_alias=True)` 按别名导出。LLM 的 schema 里用哪种命名，先在提示词与模型定义之间统一，别两边各写一套。
- **导出**：`model_dump()` 转 dict、`model_dump_json()` 转 JSON 字符串，落库与写日志都用它，不要手动拼。
- **JSON Schema 生成**：`ChapterAnalysis.model_json_schema()` 直接产出 JSON Schema——这正是给 LLM 做结构化输出（Ollama / OpenAI 兼容接口的 `response_format` 传 schema）与 Tool Calling 定义参数的原料，模型定义一份、校验与提示共用，不再手写两套。

```python
schema = ChapterAnalysis.model_json_schema()
# 传给 Ollama: options/response_format 里塞 schema，模型被约束按结构输出
# 返回后: ChapterAnalysis.model_validate_json(resp_text)  # 双保险，仍要校验
```

注意：结构化输出只是"大概率合规"，不是保证。温度、模型能力、上下文长度都会影响，**校验环节永远不能省**。

## 5. settings 与配置管理

`pydantic-settings`（v2 已拆为独立包）的 `BaseSettings` 专门管配置：从环境变量、`.env` 读值并校验类型。小说引擎的 Ollama 地址、模型名、并发数用它管，配置错启动时立刻报错：

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ollama_base: str = "http://127.0.0.1:11434/v1"
    chat_model: str = "qwen3:14b"
    embed_model: str = "embeddinggemma-2:270m"
    max_concurrency: int = 4
```

## 6. 性能与取舍

- v2 核心用 Rust 重写（pydantic-core），比 v1 快一个量级，批量抽取场景下校验不会成为瓶颈。
- 不要给内部每个小结构都套 BaseModel：校验有成本，内部可信数据用 dataclass，BaseModel 只守边界。

## 7. 常见误区

- **拿 `e.errors()` 直接抛给用户/日志了事**：LLM 场景里错误清单是重抽提示的输入，要结构化回喂。
- **在模型里写业务逻辑**：validator 里查数据库、调服务会让校验变成隐形业务层，测试和复用都痛苦；跨实体的规则放 Service 层（如矛盾检测），模型只管"形状与单实体规则"。
- **v1/v2 混写**：v2 里 `parse_obj`、class 内部 `class Config` 都是旧写法（多半还能跑但已废弃），新代码统一 `model_validate` + `ConfigDict`。
- **默认值可变对象**：`tags: list[str] = []` 在 pydantic 里其实是安全的（它会深拷贝默认值），但和 dataclass 的规则不一样，跨两个库写代码时容易记混——统一都写 `default_factory` 最稳。

## 小技巧与冷知识

【小技巧】回喂 LLM 重抽时，别直接把 `ValidationError` 的字符串拼进提示词，先调 `e.errors(include_url=False)`。默认报错里每条都带一个指向文档的 `url`，拼进提示词既占 token 又干扰模型；关掉后只剩 `loc`（出错路径）、`type`、`msg`、`input` 四个关键字段，模型一眼就能定位到是哪个字段、错在哪、原值是什么，重抽成功率更高。

【小技巧】只想校验一个列表或字典、不想为它专门建一个模型时，用 `TypeAdapter`。比如 `TypeAdapter(list[EntityChange]).validate_python(raw_list)` 可以直接校验整个列表，报错路径精确到 `2.chapter` 这样的下标；它同样支持 `validate_json`，适合“LLM 一次吐回一个数组”的批量抽取场景，比先建一个只包一层列表的顶层模型省事。

【小技巧】给函数入口加校验不用在函数体里手写一堆 `if`，用 `@validate_call` 装饰器。装饰后调用方的实参会按注解自动校验和转换，传错类型直接抛 `ValidationError`；它对 Agent 里“工具函数收 LLM 传来的参数”特别有用，等于把 pydantic 的守门员从模型入口延伸到了函数入口。

【小技巧】需要“改一个字段、其余不变地复制一份”时，别先 `model_dump()` 再重新构造，用 `model_copy(update={...})`。它返回一个新实例，只替换你指定的字段，嵌套模型默认仍是浅拷贝；想让嵌套对象也一起深拷贝再改，就加 `deep=True`。这比手写“dump → 改 dict → 重新 validate”少一次完整校验，也不会丢掉原始实例。

【小技巧】导出给外部系统时，用 `model_dump(mode="json")` 而不是默认的 `model_dump()`。默认模式会保留 Python 对象（比如 `datetime`、`UUID`、`Decimal`），直接 `json.dumps` 会报错；`mode="json"` 会先把它们转成 JSON 友好的表示（日期变 ISO 字符串等），再 `json.dumps` 就不会炸。需要同时按别名导出时，再叠加 `by_alias=True`。

【小技巧】模型里有些字段只在导出时才需要、校验时并不存在，可以用 `computed_field`。比如在模型上定义一个 `@computed_field` 修饰的 `@property`，算出派生值（如 `full_name`、统计口径的汇总值），它会自动出现在 `model_dump()` 和 JSON Schema 里，但构造模型时不用传、校验时也不会要求它，适合“原始字段进、派生字段出”的场景。

【小技巧】想让模型不可变、防止下游代码顺手改掉已校验的数据，在 `model_config` 里加 `ConfigDict(frozen=True)`。冻结后任何字段赋值都会抛错，实例也因此变得可哈希，可以放进 `set` 或当字典键；这对“校验完就当事实快照用、谁也不许改”的章节分析结果很合适。注意冻结是模型级的，改一个字段就要走 `model_copy(update=...)`。

【小技巧】对接多形态的 LLM 输出（比如事件可能是“死亡事件”也可能是“结盟事件”，字段结构各不相同），用带判别字段的联合类型：`Field(discriminator="kind")` 配合 `Union[DeathEvent, AllianceEvent]`。pydantic 会先看 `kind` 的值再决定按哪个子模型校验，报错信息也会直接说“kind 对不上哪个分支”，比不加判别器时逐个分支试错、报错堆成一团要清晰得多。

【冷知识】`ValidationError` 是 `ValueError` 的子类，所以 `except ValueError` 也能接住它。写通用异常处理时要注意顺序：如果先写了 `except ValueError`，后面专门的 `except ValidationError` 永远走不到；想区分“校验失败”和“其他值错误”，必须把 `ValidationError` 的分支写在前面。

【冷知识】模型定义本身就是可 introspect 的数据：`EntityChange.model_fields` 是一个类属性字典，键是字段名，值是 `FieldInfo`，里面有注解、默认值、约束条件。写通用工具（比如自动生成字段说明表、检查哪些字段没给默认值）时可以直接读它，不用解析源码或手写映射表。

【冷知识】pydantic 的 `BaseModel` 实例默认是不可哈希的，`hash(obj)` 会直接报错，根源是模型默认可变、Python 不给可变对象配默认哈希。上面提到的 `frozen=True` 不只是“防改”，它同时让实例获得哈希能力；反过来也可以理解：凡是需要把模型实例放进集合或当键的场景，先问一句“它冻结了吗”。

【冷知识】`extra="allow"` 时多余字段不会丢，它们存在 `model_extra` 这个字典属性里，`model_dump()` 也会把它们带上。原文建议 LLM 场景用 `forbid` 是对的，但如果你确实想“先收下模型多给的字段、事后人工复核”，`allow` + 定期检查 `model_extra` 就是一条可行的折中路：既不丢信息，又能发现模型在偷偷加什么。

【冷知识】pydantic 也有自己的 dataclass：`pydantic.dataclasses.dataclass` 装饰的 dataclass 会在构造时做校验，而标准库 `dataclasses.dataclass` 只管生成 `__init__`、不管值对不对。原文的分工是“边界用 BaseModel、内部用 dataclass”，如果某个内部结构也想顺带校验（比如配置对象），用 pydantic 的 dataclass 是个中间选项，不用把整个结构升级成 BaseModel。

【冷知识】校验器里可以拿到调用方传进来的上下文：`@field_validator` 的第二个参数是 `ValidationInfo`，它的 `context` 属性就是 `model_validate(obj, context={...})` 传进去的字典。比如“同一批章节里人物名是否已出现”这种需要外部状态的校验，可以把已有人物集合放进 `context` 传进去，而不是把状态偷偷塞进全局变量或 validator 里查库。

【冷知识】`Field` 的别名其实分两边：`alias` 管输入（校验时认哪个键），`serialization_alias` 管输出（`model_dump(by_alias=True)` 时用什么键）。对接外部系统时常遇到“输入是驼峰、输出也要驼峰，但内部字段是蛇形”的情况，两个别名分开设就能各管各的；只设 `alias` 时输出默认也跟着它走，容易在“导出给另一方”时踩坑。

【冷知识】v2 里 `model_json_schema()` 生成的 schema 对可选字段用的是 `anyOf` 结构（比如 `str | None` 会展开成 `{"anyOf": [{"type": "string"}, {"type": "null"}]}`），而不是 v1 时代常见的单一 `type`。把 schema 直接喂给对 JSON Schema 支持有限的模型或工具时，先看一眼展开后的结构，别假设它还是你手写的那种简洁形态；必要时用 `mode="serialization"` 生成另一份更贴近导出形态的 schema 对照一下。

## 面试与自测

1. 本篇把 pydantic 定位为“系统边界的守门员”，是什么意思？进了系统内部之后换什么传递？
2. 宽容模式（lax）下，`chapter: "12"` 和 `chapter: "第十二章"` 分别会怎样？想拦住 `"12"` 这种静默转换怎么开严格模式？
3. LLM 输出场景下，`extra` 为什么建议用 `forbid` 而不是默认的 `ignore`？

### 参考答案

1. 一切不可信输入——LLM 输出的 JSON、HTTP 入参、配置文件、外部 API 返回——进系统前先过一遍 pydantic，按注解做类型转换与校验，不合格直接抛 `ValidationError` 并给出结构化错误清单。进了系统内部之后换 dataclass 传递，BaseModel 只守边界。
2. `"12"` 能转就转，会被静默转成 12，不报错；对接 LLM 时这其实是优点，因为模型常把数字带引号。`"第十二章"` 转不动，直接报错，正是要拦的。想拦住 `"12"` 这类静默转换，就开严格模式：字段级 `Field(strict=True)` 或模型级 `model_config = ConfigDict(strict=True)`。
3. 默认 `ignore` 会丢弃多余字段，模型自创的字段就被悄悄吞掉了；`forbid` 让多余字段直接报错。LLM 场景建议 `forbid`，因为模型自创字段往往意味着它没按 schema 理解任务，早暴露才能早重抽。

## 8. 关联知识

- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：内部分工的一侧，边界校验通过后转 dataclass 传递
- [asyncio](开发语言/Python/asyncio.md)：批量抽取管道里，校验失败→回喂重抽是每章任务的标准收尾
- [Tool-Calling](AI与大模型/Agent/Tool-Calling.md)：工具参数 schema 与本篇的 JSON Schema 生成是同一套机制

## 9. 一句话总结

pydantic 是不可信数据的海关：LLM 输出先报关（校验 + 结构化报错 + 回喂重抽），过了关再进内部分发。
