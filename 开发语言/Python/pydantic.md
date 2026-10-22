---
知识点: pydantic 数据校验
层级: 基础
适用版本: Python 3.x
更新日期: 2026-10-22
关联:
  - Python
  - Agent
标签:
  - Python
  - 实践
---

# pydantic 数据校验

## pydantic 数据校验（LLM 输出 JSON 落库前）

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

- 小说引擎 ChapterAnalysisService 的 JSON 落库前必须过 pydantic，LLM 偶尔吐 `chapter: "第十二章"` 这种，直接进 SQLite 就是脏数据。
- v2 用 `model_validate`，别再用 v1 的 `parse_obj`（v1 已停止维护）。

