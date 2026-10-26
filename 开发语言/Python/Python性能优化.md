---
知识点: Python性能优化
层级: 进阶
适用版本: 通用
更新日期: 2026-10-26
关联:
  - Python
  - asyncio
  - Python-多进程与GIL
标签:
  - Python
  - 性能优化
  - 实践
---

# Python 性能优化

Python 性能优化只有一条纪律：先定位，再动手。没有 profile 数据的优化，大概率优化错地方。本篇按“定位 → 常见慢点 → 优化手段 → 对照数值 → 误区”的顺序讲。

## 1. 先定位：cProfile 与 timeit

- cProfile：标准库性能分析器，看函数级耗时分布。`python -m cProfile -s cumtime script.py` 按累计耗时排序输出，优先看 cumtime 高且调用次数多的函数；tottime 高说明函数自身耗时，cumtime 高说明它调用的子函数耗时。
- timeit：标准库微基准工具，适合对比“两种写法谁更快”。`python -m timeit "sum(range(100000))"` 或代码里用 `timeit.timeit(stmt, number=N)`；默认关闭 GC 做微基准，结果只用于同条件对比，不代表真实负载耗时。
- line_profiler：逐行耗时工具，能定位到具体哪一行慢。思路是“先 cProfile 定函数、再逐行定语句”；它是第三方库，本文只讲这个思路，不展开安装用法，标准库可跑的部分是 cProfile 和 timeit。
- 定位原则：先看火焰（热点函数）再看行；没有热点的“感觉慢”不值得优化。

## 2. 常见慢点

- 循环内重复计算：循环里反复调用 `len()`、查属性、编译正则。把不变的计算提到循环外，循环内只留变化的部分。
- 字符串拼接：循环里反复拼接字符串优先用 `''.join()` 收集后一次拼接；注意 CPython 对 `+=` 有原地优化，不能一概说它一定慢，但 join 的语义更稳定、可读性更好。
- 全局查找与属性查找：循环内反复访问全局变量、对象属性、方法引用时，先绑定到局部变量（如 `append = lst.append`），减少名字查找开销。
- IO 阻塞：同步 IO（网络请求、文件读写、数据库查询）等待期间 CPU 空闲，这是“程序慢”最常见的真实原因，但它不属于“代码写得慢”，解法在第 3 节的并发边界里。

## 3. 优化手段

- 用内置函数与生成器：`sum()`、`min()`、`max()`、`any()`、`all()` 是 C 实现，通常比等价的手动循环快；生成器表达式惰性产出，避免先建大中间列表。
- 缓存重复计算：`functools.lru_cache` 适合纯函数、参数可哈希、重复调用多的场景；注意 `maxsize` 与内存占用，无界缓存可能把内存吃满。
- 数据结构选型：成员检查用 `set` 不用 `list`（平均 O(1) 对 O(n)）；两端操作用 `collections.deque`；频繁按键查找用 `dict`；选型错误带来的差距往往比微调循环大几个数量级。
- 并发与异步的适用边界：CPU 密集型任务用多进程（绕开 GIL），IO 密集型任务用多线程或 `asyncio`；`asyncio` 不加速 CPU 计算，把计算密集代码包进协程只会让它继续占住事件循环。选错并发模型，优化方向就全错。

## 4. 实跑对照数值（本机实测，仅作量级参考）

下面三组对照用标准库 timeit 实跑得出，硬件与运行环境不同数值会变，看倍数关系不看绝对值。

```python
import timeit
from functools import lru_cache

# 对照 1：成员检查，list O(n) vs set O(1)
lst = list(range(100000)); st = set(lst)
t_list = timeit.timeit(lambda: 99999 in lst, number=100)
t_set = timeit.timeit(lambda: 99999 in st, number=100)
print(f"list: {t_list:.4f}s, set: {t_set:.6f}s")
# 本机实测：list 0.0979s，set 0.000011s，约 8956 倍

# 对照 2：斐波那契，朴素递归 vs lru_cache
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)
@lru_cache(maxsize=None)
def fib_c(n):
    return n if n < 2 else fib_c(n - 1) + fib_c(n - 2)
print(fib(30), fib_c(30))  # 均为 832040
t_fib = timeit.timeit(lambda: fib(30), number=1)
t_fibc = timeit.timeit(lambda: fib_c(30), number=1000)
print(f"朴素 x1: {t_fib:.3f}s, 缓存 x1000: {t_fibc:.4f}s")
# 本机实测：朴素 0.141s，缓存 1000 次 0.0001s

# 对照 3：求和，内置 sum vs 手动循环
data = list(range(100000))
t_sum = timeit.timeit(lambda: sum(data), number=100)
def manual(d):
    s = 0
    for x in d:
        s += x
    return s
t_man = timeit.timeit(lambda: manual(data), number=100)
print(f"sum: {t_sum:.4f}s, 手动循环: {t_man:.4f}s")
# 本机实测：sum 0.1203s，手动循环 0.3323s，约 2.8 倍
```

- 读法：选型（set、缓存）带来的差距是数量级，写法微调（sum）带来的差距是倍数级；优化要先抓选型类问题。

## 5. 过早优化的误区

- 没 profile 就动手：凭感觉优化的函数往往根本不是热点，改完没效果还牺牲可读性。
- 微基准当结论：timeit 里快 2 倍的写法，放到真实负载里可能只占总耗时 1%，整体收益忽略不计；优化前后都要在真实负载上复测。
- 牺牲可读性换微小收益：把清晰代码改成难懂的“技巧”写法，只为省几个百分点，通常不值得；性能收益要显著才值得牺牲可读性。
- 优化后不回归：缓存引入 stale 数据、并发引入竞态、生成器改惰性语义都可能改行为；每次优化后跑测试确认行为不变。

## 面试与自测

1. cProfile 和 timeit 分别适合定位什么问题？line_profiler 在流程里处在哪个位置？
2. 为什么成员检查要用 set 而不用 list？lru_cache 有什么使用前提和风险？
3. CPU 密集和 IO 密集任务在 Python 里分别适合什么并发模型？为什么 asyncio 加速不了 CPU 计算？

### 参考答案

1. cProfile 适合函数级定位（哪个函数 cumtime 高）；timeit 适合两种写法的微基准对比；line_profiler 处在 cProfile 之后，用于把热点函数细化到具体行，三者是“函数 → 行”的递进关系。
2. set 成员检查平均 O(1)，list 是 O(n)，数据量越大差距越大；lru_cache 要求函数是纯函数且参数可哈希，风险是无界缓存吃内存、缓存过期导致 stale 数据。
3. CPU 密集用多进程（绕开 GIL），IO 密集用多线程或 asyncio；asyncio 是单线程事件循环，CPU 计算会占住循环不让出，包进协程也不会变快，只会拖慢其他任务。
