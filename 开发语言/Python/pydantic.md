---
知识点: pydantic 数据校验
层级: 基础
适用版本: pydantic v2
更新日期: 2026-10-30
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
