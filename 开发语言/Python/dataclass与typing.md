---
知识点: dataclass 与 typing
层级: 基础
适用版本: Python 3.x
更新日期: 2026-10-22
关联:
  - Python
标签:
  - Python
  - 实践
---

# dataclass 与 typing

## dataclass + typing（告别 dict 满天飞）

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

- 小说引擎的 Entity/Relation 传来传去全用 dict 的话，字段名拼错只有运行时才知道；dataclass + 类型注解让 IDE 直接标红。
- 重一点的校验上 pydantic（见上节），轻量结构用 dataclass，零依赖。

