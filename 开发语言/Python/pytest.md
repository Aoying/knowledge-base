---
知识点: pytest 实战
层级: 基础
适用版本: Python 3.x
更新日期: 2026-10-22
关联:
  - Python
标签:
  - Python
  - 实践
---

# pytest 实战

## pytest 实战

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

- 小说引擎的抽取函数、RAG 的 chunk 函数，先写 3 个用例再写实现，TDD 在这种"输入输出明确"的场景最划算。
- `parametrize` 一行变 N 个用例，边界条件全覆盖。

