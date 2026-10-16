# Python

> 每晚 20:30 自动补充完善。

## 环境与依赖

```bash
# 虚拟环境（Windows PowerShell）
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt

# 冻结依赖
pip freeze > requirements.txt
```

## 标准库速查（转型期最常用的）

| 库 | 用途 |
|---|---|
| `pathlib` | 面向对象的路径操作，替代 `os.path` 拼字符串 |
| `json` | 读写 JSON，和 LLM 打交道天天用 |
| `datetime` | 时间处理，注意时区用 `zoneinfo` |
| `argparse` | 写命令行脚本的参数解析 |
| `logging` | 日志，别再 `print` 调试生产代码 |
| `sqlite3` | 零依赖嵌入式数据库，你的小说引擎就用它 |

## sqlite3 最小示例（小说引擎同款）

```python
import sqlite3

conn = sqlite3.connect("novel.db")
conn.row_factory = sqlite3.Row  # 按列名取数
cur = conn.cursor()
cur.execute("CREATE TABLE IF NOT EXISTS entity(id INTEGER PRIMARY KEY, name TEXT, kind TEXT)")
cur.execute("INSERT INTO entity(name, kind) VALUES(?, ?)", ("江尘", "character"))
conn.commit()

for row in cur.execute("SELECT * FROM entity WHERE kind=?", ("character",)):
    print(dict(row))
conn.close()
```

要点：`?` 占位符防注入；`row_factory` 让结果按列名访问；记得 `commit`。

## 调 Ollama 本地接口（RAG 原型同款）

```python
import requests

# 文本向量
r = requests.post("http://127.0.0.1:11434/api/embed",
                  json={"model": "embeddinggemma-2:270m", "input": "江尘挑起扁担"})
print(r.json()["embeddings"][0][:5])

# 聊天
r = requests.post("http://127.0.0.1:11434/api/chat",
                  json={"model": "qwen3:14b",
                        "messages": [{"role": "user", "content": "用一句话总结"}]})
print(r.json()["message"]["content"])
```

注意：`/api/embed` 的参数是 `input`（字符串或数组），不是 `prompt`。

## Windows 中文乱码

```python
# 读写文件永远显式指定编码
open("笔记.md", encoding="utf-8").read()

# 控制台乱码：chcp 65001 切 UTF-8，或在脚本开头加
import os
os.system("chcp 65001 >nul")
```

## 待补充

- asyncio 并发模型（小说引擎批量抽取章节时用得上）
- pydantic 数据校验（LLM 输出 JSON 落库前的校验层）
