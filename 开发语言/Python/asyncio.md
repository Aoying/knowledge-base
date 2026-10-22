---
知识点: asyncio 并发模型
层级: 进阶
适用版本: Python 3.x
更新日期: 2026-10-22
关联:
  - Python
  - Python知识点地图
标签:
  - Python
  - 并发
  - 实践
---

# asyncio 并发模型

## asyncio 并发模型（小说引擎批量抽取章节时用）

```python
import asyncio, httpx

async def extract_one(client, chapter_id):
    r = await client.post("http://127.0.0.1:11434/v1/chat/completions",
                          json={"model": "qwen3:14b", "messages": [...]})
    return chapter_id, r.json()

async def main(ids):
    async with httpx.AsyncClient(timeout=120) as client:
        sem = asyncio.Semaphore(4)  # Ollama 并发别打太高，4 左右
        async def bounded(i):
            async with sem:
                return await extract_one(client, i)
        return await asyncio.gather(*(bounded(i) for i in ids))

asyncio.run(main(range(1, 51)))
```

- IO 密集型（调 Ollama API）用 asyncio，一个线程顶几十个并发；CPU 密集型才上多进程。
- `Semaphore` 限流是保命项：Ollama 默认并发一高就排队甚至超时。
- 结构化输出别忘了 `response_format={"type":"json_object"}`，省得正则去抠。

