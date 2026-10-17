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

## 常见坑

1. **可变默认参数**：`def f(a=[])` 的 `[]` 只创建一次，跨调用共享；用 `None` + 内部初始化。
2. **浅拷贝陷阱**：`b = a[:]` / `copy.copy` 只拷一层，嵌套结构改一个两边变；用 `copy.deepcopy`。
3. **GIL 与多线程**：CPU 密集型用多线程不提速（GIL），用 `multiprocessing` 或换 PyPy/C 扩展。
4. **pip 装包装到系统 Python**：永远先建 venv（`python -m venv .venv`），再 pip；换国内源提速。
5. **文件读写编码**：Windows 下 `open()` 默认 gbk，读写 utf-8 文件必须显式 `encoding="utf-8"`。

## 疑难杂症

### 1. 循环引用导致内存泄漏
- **现象**：服务常驻进程内存缓慢上涨，gc 也回收不掉。
- **原因**：对象互相引用形成环，且其中有 `__del__` 方法（老版本 Python）或被全局容器持有，引用计数清不掉。
- **解决**：用 `weakref` 打破环；`gc.get_objects()` + `objgraph` 定位泄漏点；长期运行的服务定期打内存快照对比。

### 2. 多进程 pickle 失败：Can't pickle local object
- **现象**：`multiprocessing` 传函数/对象报 pickling 错误。
- **原因**：pickle 只能序列化顶层定义的函数/类，闭包、lambda、局部类传不过去。
- **解决**：把函数提到模块顶层；复杂对象用 `dill` 替代 pickle；或改用线程+队列绕开。

### 3. 依赖冲突：装 A 坏 B
- **现象**：`pip install` 新包装完，旧包报 `ImportError` 或版本不兼容。
- **原因**：pip 默认不做严格的依赖求解，新包把共享依赖升级/降级了。
- **解决**：`pip install` 前用 `pipdeptree` 看依赖树；大项目锁版本（`pip freeze > requirements.txt`）；隔离用 venv，别往 base 环境装。
