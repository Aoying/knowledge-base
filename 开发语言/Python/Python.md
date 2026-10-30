---
知识点: Python
层级: 基础
适用版本: Python 3.x
更新日期: 2026-10-31
关联:
  - Python知识点地图
  - asyncio
  - pydantic
  - 正则表达式
标签:
  - Python
  - 实践
---

# Python

> 每晚 20:30 自动补充完善。

## 知识点拆分索引

本篇为 Python 语言基础与总索引，以下知识点已拆为独立文档：

- [asyncio 并发模型](开发语言/Python/asyncio.md)
- [pydantic 数据校验](开发语言/Python/pydantic.md)
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)
- [pytest 实战](开发语言/Python/pytest.md)
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)
- [FastAPI](开发语言/Python/FastAPI.md)
- [Python 打包与环境管理](开发语言/Python/Python-打包与环境管理.md)
- [装饰器](开发语言/Python/装饰器.md)
- [生成器](开发语言/Python/生成器.md)

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

## 基础知识补强

### 核心概念与术语

- **动态类型**：变量不用声明类型，类型绑在对象上；同一个变量先后装 int、str 都合法，但可读性会变差。
- **一切皆对象**：数字、字符串、函数都是对象，都有类型、可以被赋值和传递。
- **强类型**：`"1" + 1` 直接报错，不会偷偷把数字转成字符串，类型转换要自己写 `int()`、`str()`。
- **基本标量**：`int`（任意精度整数）、`float`（浮点，有精度误差）、`str`（不可变字符串）、`bool`（True/False）、`None`（表示"无"，类似其他语言的 null）。
- **四大容器**：`list` 有序可变、`tuple` 有序不可变、`dict` 键值映射、`set` 无序不重复集合。
- **可变与不可变**：list、dict、set 可变（原地改）；int、str、tuple 不可变（"改"实际是生成新对象）。这是很多坑的总源头。
- **序列**：能按下标和切片访问的一类对象，list、tuple、str 都是序列，下标从 0 开始，负数从末尾数。
- **迭代**：凡是能被 `for` 逐个取出元素的对象都可迭代；迭代器是"一次取一个、取完抛 StopIteration"的对象。
- **缩进即语法**：Python 用缩进划分代码块，没有大括号；同级缩进必须一致，一般用 4 个空格。
- **引用语义**：赋值 `b = a` 只是多贴一个标签指向同一对象，不复制内容；对象可变时两边互相影响。
- **真值判断**：空容器、空字符串、数字 0、None 都是假（falsy），其余绝大多数对象为真。
- **模块与包**：一个 `.py` 文件就是一个模块；带 `__init__.py`（新版可省）的目录是包；`import` 是把模块加载进来用。

### 基础语法与用法

```python
# 变量与基本类型：不用写类型，直接赋值
age = 18                 # int
price = 9.9              # float
name = "ao"              # str
ok = age >= 18           # bool: True
nothing = None           # 空值

# f-string：拼字符串最常用写法，花括号里写表达式
print(f"{name} 今年 {age} 岁, 明年 {age + 1} 岁")

# 类型自查与转换
print(type(age))         # <class 'int'>
n = int("42")            # 字符串转整数，内容非法会抛 ValueError
s = str(42)              # 转成 "42"
```

```python
# 控制流：注意冒号和缩进，缺一不可
score = 85
if score >= 90:
    level = "A"
elif score >= 60:
    level = "及格"
else:
    level = "不及格"

for i in range(3):       # range(3) 产出 0, 1, 2，不含终点
    print(i)

n = 3
while n > 0:
    n -= 1

# 三元表达式：一行写简单的二选一
label = "成年" if age >= 18 else "未成年"
```

```python
# 四大容器基础操作
nums = [3, 1, 2]                 # list
nums.append(4)                   # 末尾加
nums.sort()                      # 原地排序，返回 None
first = nums[0]                  # 下标取
last_two = nums[-2:]             # 切片：左闭右开

point = (10, 20)                 # tuple，不可改，可当 dict 的键
x, y = point                     # 解包赋值

user = {"name": "ao", "age": 5}  # dict
print(user.get("city", "未知"))  # get 带默认值，键不存在不报错
user["city"] = "深圳"            # 加/改键

tags = {"rag", "agent", "rag"}   # set 自动去重 -> {"rag", "agent"}
print("rag" in tags)             # 成员判断，set 里很快
```

```python
# 函数：默认值、关键字参数、返回值
def greet(name, prefix="你好"):
    return f"{prefix}, {name}"

print(greet("ao"))                      # 用默认值
print(greet("ao", prefix="早上好"))     # 关键字参数，不看位置

def divmod2(a, b):
    return a // b, a % b     # 多返回值本质是返回一个 tuple

q, r = divmod2(17, 5)        # 解包接收 -> q=3, r=2

# 推导式：一行生成容器（别写太复杂，超过一层嵌套就拆回 for）
squares = [x * x for x in range(5)]            # [0, 1, 4, 9, 16]
evens = [x for x in range(10) if x % 2 == 0]   # 带过滤
name_len = {w: len(w) for w in ["ai", "rag"]}  # 字典推导
```

```python
# 异常与文件：with 自动关文件，是固定写法
try:
    with open("笔记.md", encoding="utf-8") as f:
        text = f.read()
except FileNotFoundError:
    text = ""                # 文件不存在时给兜底值
except UnicodeDecodeError as e:
    print("编码不对:", e)    # 具体异常具体抓，别裸 except

# 类的最简形态
class Chapter:
    def __init__(self, title, words=0):   # 构造器，self 是实例本身
        self.title = title
        self.words = words

    def is_long(self):                    # 实例方法第一个参数永远是 self
        return self.words > 5000

ch = Chapter("第一章", 6200)
print(ch.title, ch.is_long())             # 属性访问不用 get 前缀
```

### 关键机制速览

- **名字是标签不是盒子**：变量是贴在对象上的名字，重新赋值只是换标签，对象本身没被复制。
- **相等与同一**：`==` 比值是否相等，`is` 比是不是同一个对象；判断 None 永远写 `x is None`。
- **小整数与字符串驻留**：CPython 会缓存小整数和部分短字符串，所以偶尔 `is` 也为真，别拿它当相等判断用（以官方文档为准）。
- **切片生成新对象**：`a[:]` 是浅层新列表，原列表增删不影响切片，但元素对象本身仍是共享的。
- **函数参数传递**：传的是对象引用；函数内给可变对象做原地修改，外面能看见；重新赋值则看不见。
- **作用域 LEGB**：查找名字顺序为局部、嵌套、全局、内置；函数内给全局变量赋值要先 `global` 声明，但业务代码少用。
- **迭代协议**：`for` 背后先调 `iter()` 拿迭代器，再反复 `next()`；自定义类实现 `__iter__` 就能被 for 用。
- **生成器**：带 `yield` 的函数返回生成器，用到才算，省内存；大文件逐行处理、惰性管道靠它。
- **真值与短路**：`and` 返回第一个假值或最后一个值，`or` 返回第一个真值，所以有 `x = val or "默认"` 这种惯用法。
- **字符串不可变**：拼接循环里反复 `s += part` 可能很慢，大量拼接用 `"".join(parts)`。
- **导入执行**：`import` 时模块顶层代码会执行一遍，所以可执行脚本入口写在 `if __name__ == "__main__":` 下面。

### 新手易错点

- **缩进混用空格和 Tab**：肉眼看不出，解释器报 IndentationError 或逻辑错位，全文件统一 4 空格。
- **拿 `is` 比字符串和大数**：小范围"碰巧"为真，换个值就假；比内容一律用 `==`。
- **修改正在遍历的列表**：`for x in lst: lst.remove(x)` 会跳过元素，先遍历副本 `lst[:]` 或生成新列表。
- **dict 遍历时改键**：迭代中增删键抛 RuntimeError，先把键收集成 list 再动手。
- **整数除以得到浮点**：`/` 永远是浮点除法（`4 / 2 == 2.0`），要整除用 `//`，别从其他语言习惯里默认。
- **默认参数写了当前时间**：`def f(t=datetime.now())` 只在定义时求值一次，所有调用共用那个旧时间；写 None 再进函数补。
- **命名遮蔽内置名**：变量叫 `list`、`id`、`type` 后，本作用域里就再也用不了这些内置函数了。
- **浮点相等判断**：`0.1 + 0.2 == 0.3` 为 False，金额用整数分或 Decimal，别直接比。
- **except 裸抓一切**：`except:` 连拼写错误、KeyboardInterrupt 一起吞，排障无从下手；至少抓具体异常并打印。
- **切片越界不报错**：`lst[10:20]` 对短列表只返回已有部分，逻辑错误会静默通过，别指望它帮你查界。

## 小技巧与冷知识

【小技巧】装包、跑模块时用 `python -m` 而不用裸命令：`python -m pip install requests` 保证装进的就是当前这个解释器对应的环境，裸 `pip` 可能指向系统里另一个 Python。`python -m json.tool 文件.json`、`python -m http.server` 这类“模块当命令跑”的写法也是同一招——`python` 选解释器，`-m` 选模块，两个动作一次定死，不会出现“装到了 A 环境、跑在 B 环境”的错位。

【小技巧】往 JSON 里写中文时加 `ensure_ascii=False`：`json.dumps({"name": "江尘"}, ensure_ascii=False)` 直接出中文，默认 `ensure_ascii=True` 会把中文转成 `\u6c5f\u5c18` 这种转义，文件能读、但人眼读不了。调 LLM 接口、写中文日志文件时尤其要开这个参数；注意它只影响序列化输出，解析时两边都认，不用两头都改。

【小技巧】`[[]] * 3` 里的三个内层列表是同一个对象：乘法复制的是引用不是内容，往 `a[0]` 里 append 一个元素，三个子列表一起变。`a = b = []` 也是同一回事——两个名字贴在同一个列表上。想“三个独立空列表”用推导式 `[[] for _ in range(3)]`，每次循环新建一个；`[0] * 3` 没问题是因为整数不可变，共享也没人能改。

【小技巧】字典合并用 `|`（Python 3.9+）：`{"a": 1} | {"b": 2}` 得到新字典，`d |= {"b": 2}` 是原地合并，比 `{**a, **b}` 更直白。`d.get(k)` 取不到只返回默认值、不写进字典；想“取不到就顺手存一个”用 `d.setdefault(k, [])` 或 `collections.defaultdict`——“只读”和“读不到就建”分工不同，别用 `get` 假装写回。

【小技巧】并行遍历多个序列时给 `zip` 加 `strict=True`（Python 3.10+）：默认 `zip` 按最短的那个停，两个列表长度差一个、数据错位了也不会报错；`zip(a, b, strict=True)` 会在长度不等时直接抛 `ValueError`。对齐处理“成对出现”的数据（章节列表和字数列表、问题列表和答案列表）时，宁可让它在长度不对时炸掉，也别让错位数据悄悄进库。

【小技巧】生成器不能切片、取不到元素会抛 `StopIteration`：`gen[0:3]` 直接报 TypeError，想取前 N 个用 `itertools.islice(gen, 3)`；`next(gen)` 在耗尽时抛异常，写 `next(gen, 默认值)` 耗尽就返回默认值。`for` 循环内部已经帮你处理了 StopIteration，自己手动 `next` 时才要记得兜底。

【小技巧】`while` 循环里读一行、判一行时用海象运算符 `:=`：`while (line := f.readline()): 处理(line)` 把“读”和“判断”合成一行，不用先读一次、循环里再读一次。`:=` 是“赋值并返回这个值”，适合“先算出来、马上就要判断它”的场合；别在复杂表达式里滥用，一行里出现两个海象就该拆回普通写法。

【小技巧】`for...else` 和 `try...else` 的 `else` 只在“没提前退出”时跑：`for` 循环没遇到 `break` 正常结束才进 `else`，适合“找了一圈没找到”的逻辑；`try` 块没抛异常才进 `else`，适合把“只有成功时才做的事”和“可能抛错的事”分开。`else` 不是“否则”，是“一切顺利才执行”，读成这个意思就不会写错位置。

【小技巧】`sorted`、`min`、`max` 的 `key=` 参数比先加工数据再排序省事：`sorted(chapters, key=lambda c: c["words"], reverse=True)` 直接按字数排，不用先建一个“字数-章节”的中间表；`min([], default=0)` 能在空序列上返回默认值而不是抛 `ValueError`。`key` 只在比较时调用一次、结果会缓存，不用担心“每比较一次就算一次”。

【小技巧】脚本里相对导入报错时，从项目根目录用 `python -m 包.模块` 跑：直接 `python 子目录/脚本.py` 时，`sys.path[0]` 是脚本所在目录，包的相对导入找不到父包；`python -m novelTool.main` 从项目根启动，`sys.path[0]` 是当前工作目录，包结构一次到位。写死 `sys.path.insert` 是退路，改启动方式才是正解。

【小技巧】调试时用内置的 `breakpoint()` 而不是散落的 `print`：`breakpoint()`（Python 3.7+）直接进 pdb，`n` 单步、`p 变量名` 看值、`c` 继续，比加一堆 `print` 再删干净利索得多。想全局关掉它设环境变量 `PYTHONBREAKPOINT=0`，线上跑脚本时不会被某个忘删的断点卡住；f-string 里写 `f"{变量名=}"`（3.8+）能同时打出名字和值，是轻量版的调试打印。

【小技巧】计时用 `time.perf_counter()`，别用 `time.time()`：`perf_counter` 是单调时钟，只往前走，适合量一段代码跑了多久；`time.time()` 是墙上时钟，系统时间被 NTP 回拨时会倒流，量出来的耗时可能是负数。`time.sleep` 的精度也别指望到毫秒级，它只保证“至少睡这么久”，醒来的具体时刻由调度器决定。

【小技巧】`assert` 只做开发期的自检，别拿它做生产校验：`python -O`（优化模式）会把所有 `assert` 语句直接删掉，靠它校验用户输入、校验接口返回，线上跑 `-O` 时校验就全没了。需要一直成立的检查用 `if ...: raise ValueError(...)` 显式写出来；`assert` 只适合“开发时我认为这里一定成立、错了就立刻暴露”的内部假设。

【冷知识】`round(2.5)` 的结果是 `2` 不是 `3`：Python 的 `round` 用的是“银行家舍入”（四舍六入、五成双），`round(2.5) == 2`、`round(3.5) == 4`，取最近的偶数。同理 `-7 // 2` 是 `-4` 而不是 `-3`，`//` 是向下取整；`-7 % 3` 是 `2`，余数的符号跟除数。这三个都是“数学定义和直觉差半格”的地方，写数值逻辑时先在 REPL 里验一下。

【冷知识】Python 的名字来自喜剧团体 Monty Python，不是蛇：作者 Guido van Rossum 在 1991 年发布它时正好在看《Monty Python's Flying Circus》，就借了这个名字；语言吉祥物是两条蛇只是后来的梗。想看这门语言的设计哲学，在解释器里敲 `import this`，会打出《Zen of Python》（优美胜于丑陋、显式胜于隐式、简单胜于复杂……）全文。

【冷知识】`True` 和 `False` 在 Python 2 里是可以被重新赋值的：`True, False = False, True` 在 Python 2 里合法，Python 3 才把它们变成真正的关键字；`print` 也从 Python 2 的语句变成了 Python 3 的函数。Python 2 已于 2020 年 1 月 1 日停止维护，老代码迁移时这两处是必改项，不是风格问题。

【冷知识】`range(10**9)` 几乎不占内存：`range` 返回的是惰性序列，只存起止和步长，不会真的生成十亿个整数；`len(range(10**9))` 秒回。反过来，`int` 转字符串在 Python 3.11+ 有 4300 位的上限（`sys.get_int_max_str_digits()`），超长整数 `str()` 会直接报错——这是防“超大整数转字符串拖垮服务”的安全限制，不是 bug。

## 面试与自测

1. `def f(a=[])` 的 `[]` 为什么会在多次调用之间共享？正确的写法是什么？
2. 本篇 sqlite3 最小示例里，`?` 占位符、`row_factory`、`commit` 各起什么作用？
3. 调 Ollama 的 `/api/embed` 时，传参为什么要用 `input` 而不是 `prompt`？

### 参考答案

1. `[]` 只在函数定义时创建一次，之后每次调用共用同一个对象，所以跨调用共享。正确写法是用 `None` 做默认值，在函数内部再初始化一个新的列表。
2. `?` 占位符防注入；`row_factory = sqlite3.Row` 让结果可以按列名取数；`commit` 提交写入，不提交数据不会落库。
3. 因为 `/api/embed` 的参数约定就是 `input`（字符串或数组），不是 `prompt`；按错名字传参调不通，原文特意把这一点标为注意点。
