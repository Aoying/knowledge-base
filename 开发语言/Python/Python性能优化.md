---
知识点: Python性能优化
层级: 进阶
适用版本: 通用
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】cProfile 别每次都重跑全程序：`python -m cProfile -o out.prof script.py` 把结果存下来，再用标准库 `pstats` 反复看——`Stats("out.prof").sort_stats("cumulative").print_callers()` 能看到“是谁调了这个热点函数”，只看 cumtime 最高的函数有时只能看到症状，看调用链才能找到根因。只想测关键一段时，用 `cProfile.Profile()` 的 `enable()`/`disable()` 把分析范围收窄，profile 本身的开销也不会污染全程数据。

【小技巧】timeit 对比时用 `repeat` 取最小值，别只跑一次：`timeit.repeat(stmt, repeat=5, number=1000)` 跑五轮取最小的那轮，系统抖动、后台进程抢 CPU 都会让单次结果偏高，最小值最接近“这段代码本身的真实耗时”。原文第 4 节的对照就是这个思路的落地——看倍数关系不看绝对值，单次数字本身没意义。

【小技巧】自己写计时别用 `time.time()`，用 `time.perf_counter()`：`perf_counter` 是单调时钟、分辨率最高，不受系统时间被校准、被手动改的影响；`time.time()` 是墙上时钟，系统时间一跳，计时结果就跟着跳。测一段代码耗时、算请求延迟都用 `perf_counter`，`time.time()` 只适合“现在是几点”这种要和真实时间对齐的场景。

【小技巧】想确认“局部变量为什么更快”，用 `dis` 看字节码：`dis.dis(函数)` 里全局变量对应 `LOAD_GLOBAL`（要查全局字典），局部变量对应 `LOAD_FAST`（按下标直接取），方法引用先绑到局部变量再调用，省的就是这一层字典查找。优化前先看一眼字节码，猜“这段代码慢在哪”就有了证据，而不是凭感觉。

【小技巧】lru_cache 调优先看 `cache_info()` 再调 `maxsize`：`fn.cache_info()` 直接给出 hits/misses，命中率低说明缓存没起作用，可能是参数变化太多、缓存根本复用不上，调大 maxsize 也白搭。方法上慎用 `lru_cache`——缓存的 key 里含 `self`，实例会被缓存一直引用着、无法释放；实例属性只算一次的场景用 `functools.cached_property`（Python 3.8+），它随实例一起释放。

【小技巧】命中率很高时，用 try/except 代替“先 in 检查再取值”：try 块在不抛异常时几乎没有额外开销，且只查一次字典；先 `if k in d` 再 `d[k]` 是查两次。但这只是少一次查找的量级，实测里两者经常打平，别为它改可读性；命中率低时异常本身的开销反而更大，那时先检查更划算——选哪种写法看命中率，不看教条。

【小技巧】推导式比等价的“循环 + append”快：`[x * x for x in data]` 的循环体在 C 层跑，`result.append(...)` 每次都要经过一次方法调用和属性查找。字典推导、集合推导同理；本机实测一万个元素的平方，推导式比循环 append 快约两成。性能和可读性在这里不冲突——推导式通常还更短。

【小技巧】`sorted`、`max`、`groupby` 的 key 别写 lambda，用 `operator.itemgetter` / `attrgetter`：它们是 C 实现的取字段函数，比 lambda 少一层 Python 函数调用，本机实测五千条记录排序，`key=itemgetter(0)` 比 `key=lambda t: t[0]` 快约四分之一。字段名写在 itemgetter 的参数里，意图和 lambda 一样直白。

【小技巧】大量小对象先想 `__slots__`：`__slots__ = ("x", "y")` 让实例不再建 `__dict__`，每个实例省一份字典的内存，成千上万个对象时内存和创建速度都有收益。代价是实例不能再随意挂新属性，子类想继续省也得自己声明 `__slots__`；普通业务类没必要上，数据类、缓存条目这类“字段固定、数量大”的类最适合。

【小技巧】已排序的列表上查元素用 `bisect`，别用 `in` 线性扫：`bisect_left(a, x)` 是二分查找，`in` 是从头扫到尾，列表一长差距就是 O(log n) 对 O(n)。`bisect` 还能直接给出插入位置，维护有序列表时“查 + 插”一次到位；但它只在列表已经有序时才有效，无序列表先 `sort` 再用它。

【小技巧】数值大循环先想 NumPy 向量化，纯 Python 循环是最后手段：`arr * 2 + 1` 整列在 C 层一次算完，等价的 Python 循环要逐个元素走解释器，数据量越大差距越大。但向量化不是免费的——NumPy 数组和 Python 列表来回转换有成本，小数据量、只算一两次时别急着引 NumPy，先 profile 确认循环真是热点再换。

【小技巧】想看生产进程的热点又不想改代码，用采样型 profiler（比如 py-spy）：它直接 attach 到正在跑的进程上定时采样调用栈，不用插桩、不用重启，火焰图的形状和 cProfile 是一致的。cProfile 适合开发环境精确定位，采样 profiler 适合“线上慢、本地复现不了”的场景，两者互补。

【小技巧】性能问题常是内存问题，先用 `tracemalloc` 看分配：`tracemalloc.start()` 后 `get_traced_memory()` 能拿到当前与峰值内存，`take_snapshot().statistics("lineno")` 直接定位“哪行代码分配最多”。频繁分配大临时对象会触发 GC，GC 停顿摊到每次调用上就是“代码慢”；定位到分配大户再决定是复用、改生成器还是换数据结构。

【冷知识】cProfile 的前身 `profile` 是纯 Python 实现的：标准库里 `profile` 和 `cProfile` 接口一样，但 `profile` 是 Python 写的，分析开销比被测代码本身还大，基本只剩教学意义；`cProfile` 是 C 实现的，才是日常用的那个。名字里的 c 就是 C 的意思。

【冷知识】Python 3.12 起 `sum()` 对浮点数换了求和算法：改用 Neumaier 求和，同样一列浮点数求和，3.12 的结果比 3.11 更准，代价是几乎感觉不到的额外开销。同一个函数、同一份输入，升级解释器后算出的小数点后几位可能变了——那是精度变好了，不是 bug。

【冷知识】Python 3.11 的提速来自“自适应专门化解释器”：Faster CPython 项目让解释器在运行时给热点字节码换上专门化版本，官方基准测试平均快约 1.25 倍。也就是说，同一份代码从 3.10 升到 3.11，不改一行就可能快两成多——升级解释器本身有时就是最划算的一次优化。

【冷知识】`functools.cache` 就是 `lru_cache(maxsize=None)` 的简写：Python 3.9 起可以用 `@cache` 代替 `@lru_cache(maxsize=None)`，两者行为完全一致。无界缓存省掉了“设多大”的纠结，但也省不掉原文说的风险——缓存永不淘汰，键一多照样吃满内存。

## 面试与自测

1. cProfile 和 timeit 分别适合定位什么问题？line_profiler 在流程里处在哪个位置？
2. 为什么成员检查要用 set 而不用 list？lru_cache 有什么使用前提和风险？
3. CPU 密集和 IO 密集任务在 Python 里分别适合什么并发模型？为什么 asyncio 加速不了 CPU 计算？

### 参考答案

1. cProfile 适合函数级定位（哪个函数 cumtime 高）；timeit 适合两种写法的微基准对比；line_profiler 处在 cProfile 之后，用于把热点函数细化到具体行，三者是“函数 → 行”的递进关系。
2. set 成员检查平均 O(1)，list 是 O(n)，数据量越大差距越大；lru_cache 要求函数是纯函数且参数可哈希，风险是无界缓存吃内存、缓存过期导致 stale 数据。
3. CPU 密集用多进程（绕开 GIL），IO 密集用多线程或 asyncio；asyncio 是单线程事件循环，CPU 计算会占住循环不让出，包进协程也不会变快，只会拖慢其他任务。
