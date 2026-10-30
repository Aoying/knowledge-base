---
知识点: Python-NumPy与Pandas基础
层级: 基础
适用版本: 通用
更新日期: 2026-10-31
关联:
  - Python
  - SQLAlchemy
  - Python性能优化
标签:
  - Python
  - 实践
---

# Python-NumPy与Pandas基础

## 1. 核心概念

NumPy 和 Pandas 是 Python 做数据处理的两个底座，分工一句话：

- **NumPy**：多维数组 `ndarray`，数值计算一次对整组做，不用写循环。
- **Pandas**：带行列标签的表格 `DataFrame`（一列就是 `Series`），做筛选、分组、连接、缺失处理这些表操作。

为什么不用纯 Python 的 list 和 dict 硬算：循环逐个处理在数据量上来后又慢又啰嗦，而且对齐、缺失、分组这些逻辑全得手写；NumPy/Pandas 把它们变成了现成的、按整列算的操作。对应关系：

| 纯 Python | NumPy / Pandas |
|---|---|
| list | ndarray（一维数组）、Series（带索引的一列） |
| list of dict | DataFrame（带列名的二维表） |
| for 循环逐个算 | 向量化：一次对整列算 |
| 手写分组统计 | `groupby` |

适用场景：CSV/日志/表格数据的清洗统计、特征预处理、一切"先算清楚再入库/再建模"的环节。持久化到数据库那一步见 [SQLAlchemy](开发语言/Python/SQLAlchemy.md)，这里只管算表。

## 2. 基础知识

### ndarray：创建、形状、索引

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5, 6])
print(a.shape)        # (6,)：一维，6 个元素
print(a.dtype)        # 整数类型（具体位宽随平台）

b = a.reshape(2, 3)   # 改形状，通常是视图，不复制数据
print(b)
# [[1 2 3]
#  [4 5 6]]

mask = a > 3
print(a[mask])        # [4 5 6]：布尔掩码筛选，条件写在方括号里

print(a * 2)          # [ 2  4  6  8 10 12]：向量化，整组一起算
print(a + 10)         # 广播：标量自动扩展到每个元素
```

要点：`shape` 是各维长度，`dtype` 是元素类型（int/float/bool 等）；运算默认对整组做，不写循环；`reshape` 只是换个看法，改 `b` 会影响 `a`（视图语义，见第 3 节）。

### DataFrame 与 Series：创建、筛选、loc/iloc

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["北京", "上海", "北京", "广州"],
    "product": ["手机", "手机", "耳机", "手机"],
    "amount": [1200, 1500, 300, 900],
})
print(df[df["amount"] > 500])                  # 布尔掩码：行筛选
print(df.loc[df["city"] == "北京", "amount"])  # loc：按标签/条件取，行条件 + 列名
print(df.iloc[0])                              # iloc：按位置取，第 0 行
```

- `df["amount"]` 取一列，得到 Series；`df[["city", "amount"]]` 取多列，仍是 DataFrame。
- **loc 按标签/条件，iloc 按位置**：默认整数索引时两者看起来一样，一旦索引被改过（比如 `set_index` 后），混用立刻出错。
- Series 有自己的索引：`pd.Series([10, 20, 30], index=["a", "b", "c"])`，按索引对齐是 Pandas 很多行为的根源。

### 分组聚合：groupby

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["北京", "上海", "北京", "广州"],
    "amount": [1200, 1500, 300, 900],
})
result = df.groupby("city")["amount"].sum()
print(result)
# city
# 上海    1500
# 北京    1500
# 广州     900
# Name: amount, dtype: int64
```

`groupby("city")` 先按 city 分组，再对选中的列聚合（sum/mean/count/max/min）。习惯：聚合前先选中列（`["amount"]`），避免对无关列也聚合一遍；groupby 默认按分组键排序，所以结果里上海排在北京前面。

### 缺失值：先数，再处理

缺失在 Pandas 里是 `NaN`（浮点列里出现缺失，整列会变 float，这是正常的）。处理三件套：

- `isna()`：判断哪格缺失（`NaN` 不等于任何值，包括自己，见第 7 节）。
- `dropna()`：删掉含缺失的行/列。
- `fillna()`：填值（0、均值、前值等），填什么取决于业务含义，不能无脑填 0——金额缺失填 0 和"没卖出"是两回事。

## 3. 进阶知识

### 向量化与广播

向量化是性能的来源：`a * 2` 在底层整组算，而不是 Python 层逐个循环。广播是向量化的配套规则：形状不同的数组运算时，小的一方自动扩展——标量加数组最常见；两个数组广播时从尾部维度对齐，长度为 1 的维可扩展，对不齐就报错。记不住规则时先看 `shape`，对不齐先 `reshape` 再算。

### 视图 vs 副本：最值得单拎出来的机制

- **视图**：只换看法，不复制数据。`reshape`、切片 `a[0:2]` 属于视图，改视图会动原数据；布尔掩码索引与花式索引返回的是副本。
- **副本**：`copy()`、花式索引（整数数组索引）等是真复制，改副本不影响原数据。
- 规则记不住时直接 `arr.copy()` 求稳；分不清时用 `np.shares_memory(a, b)` 验证。

DataFrame 侧的同款坑是链式赋值（`df[条件]["列"] = 值`），见第 7 节：一次 `loc` 到位是正解。

### 连接：merge

```python
import pandas as pd

orders = pd.DataFrame({"city": ["北京", "上海"], "amount": [1200, 1500]})
regions = pd.DataFrame({"city": ["北京", "上海", "广州"], "region": ["华北", "华东", "华南"]})
merged = orders.merge(regions, on="city", how="left")
print(merged)
#    city  amount region
# 0  北京    1200     华北
# 1  上海    1500     华东
```

`how` 对应 SQL 的 JOIN 语义：left（左表全留）、inner（只留交集）、outer（全留）。连接前先确认键有没有重复值——键重复会让行数爆炸式增长（笛卡尔积），先 `drop_duplicates` 或先聚合再连。

### apply：能不用就不用

`apply` 是把 Python 函数逐行/逐列套上去，本质还是循环，只是写法像向量化。优先顺序：现成向量化方法（加减乘除、字符串方法、条件 `where`）> `apply`。`apply` 只留给"确实没有现成方法"的逻辑。

## 4. 与纯 Python / SQL 的对照

| 操作 | SQL | pandas |
|---|---|---|
| 分组求和 | `SELECT city, SUM(amount) FROM t GROUP BY city` | `df.groupby("city")["amount"].sum()` |
| 筛选 | `WHERE amount > 500` | `df[df["amount"] > 500]` |
| 连接 | `JOIN ... ON` | `df.merge(other, on="city")` |
| 去重 | `DISTINCT` | `df.drop_duplicates()` |
| 排序取前 N | `ORDER BY x DESC LIMIT 5` | `df.sort_values("x", ascending=False).head(5)` |
| 缺失填充 | `COALESCE(amount, 0)` | `df["amount"].fillna(0)` |

纯 Python 对照：`sum(x for x in lst if x > 500)` 对应布尔掩码 + `sum`；`collections.Counter` 分组计数对应 `value_counts()`；字典按 key 累加对应 `groupby.sum()`。思路一样，Pandas 是把"循环 + 临时容器"收进了现成操作。

## 5. 实战应用：读 CSV、清洗、分组出结果

```python
import io
import pandas as pd

csv = """city,product,amount
北京,手机,1200
上海,手机,
北京,耳机,300
上海,耳机,250
广州,手机,900
"""
df = pd.read_csv(io.StringIO(csv))
print(df["amount"].isna().sum())   # 1：先数缺失，再决定怎么处理

df["amount"] = df["amount"].fillna(0)
result = df.groupby("city")["amount"].sum().sort_values(ascending=False)
print(result)
# city
# 北京    1500.0
# 广州     900.0
# 上海     250.0
# Name: amount, dtype: float64
```

流程固定四步：读入（`read_csv`）→ 看缺失（`isna().sum()`）→ 处理缺失（这里填 0 是示例，真实业务先确认含义）→ 分组聚合排序。`read_csv` 常用参数：`usecols`（只读需要的列）、`dtype`（定死列类型，防推断跑偏）、`parse_dates`（日期列直接解析）。

## 6. 性能优化

- **向量化优先**：整列运算代替 `iterrows`/`itertuples` 循环；`iterrows` 是最慢的路，只在确实需要逐行且无替代时用。
- **读入时收窄**：`usecols` + `dtype` 在读入阶段就把量压下来，比读全表再筛选省得多。
- **重复值少的字符串列转 category**：`astype("category")`，省内存、分组更快。
- **别在循环里拼接 DataFrame**：先把每批结果收进 list，最后一次 `pd.concat`；循环里 `concat` 是平方级开销。
- 更大体量的优化（分块读 `chunksize`、并行、换引擎）见 [Python性能优化](开发语言/Python/Python性能优化.md)。

## 7. 常见误区

- **链式赋值**：`df[条件]["列"] = 值` 可能只改了中间副本，原表不动，还会报 `SettingWithCopyWarning`。一次 `loc` 到位：

```python
import pandas as pd

df = pd.DataFrame({"city": ["北京", "上海"], "amount": [100, 200]})
# 错：链式赋值，可能只改了副本，原表不动（还会报 SettingWithCopyWarning）
df[df["city"] == "北京"]["amount"] = 0
# 对：一次 loc 到位
df.loc[df["city"] == "北京", "amount"] = 0
print(df)
#    city  amount
# 0  北京       0
# 1  上海     200
```

- **用 `==` 判断缺失**：`NaN` 不等于任何值，包括自己，`== np.nan` 永远是 False。判断缺失只用 `isna()`：

```python
import pandas as pd

s = pd.Series([1.0, float("nan"), 3.0])
print(s == float("nan"))  # 全是 False：NaN 不等于任何值，包括自己
print(s.isna())           # 0 False、1 True、2 False
```

- **groupby 后索引变了**：聚合结果的分组列变成了索引，后续想当普通列用要 `reset_index()`；或者聚合时加 `as_index=False`。
- **把 DataFrame 当 dict 用**：`df["不存在的列"]` 报错，`df.不存在的属性` 也一样；列名有空格/中文时只用方括号，别用点号。
- **无脑 `fillna(0)`**：填 0 会改变均值、求和与"缺失率"本身；先确认缺失含义（没发生 vs 没记录），再选删除、填值或插值。
- **inplace 的错觉**：多数方法默认返回新对象，`df.dropna()` 不加 `inplace=True` 时原表不变；新版风格更推荐赋值写法（`df = df.dropna()`），可读且不踩坑。

## 小技巧与冷知识

【小技巧】取单个格子用 `at`/`iat`，别用 `loc`/`iloc`：`df.at[行标签, 列名]` 和 `df.iat[行号, 列号]` 是专门为“取一个标量”设计的，省掉了 `loc`/`iloc` 切片、构造中间对象的开销。循环里逐格读写时这个差距会累积；取一整行、一整列才用 `loc`/`iloc`，取一个值就用 `at`/`iat`。

【小技巧】条件赋值用 `np.where`，别写嵌套 `apply`：`np.where(df["amount"] > 1000, "大单", "小单")` 一行按条件生成新列，底层是整列向量化运算。多档条件就嵌套 `where` 或用 `pd.cut` 分箱，比 `apply(lambda row: ...)` 逐行判断快得多；`where` 的三个参数分别是条件、为真取什么、为假取什么，读起来就是一句 if-else。

【小技巧】`merge` 之前先用 `validate` 自检键的唯一性：`merge(..., validate="one_to_one")` 会在键重复时直接抛 `MergeError`，而不是悄悄把行数乘出来。常见的四种取值是 `one_to_one`、`one_to_many`、`many_to_one`、`many_to_many`，按业务里“键该不该唯一”选一个写死；连接后行数异常时先查 `validate`，别先查业务逻辑。

【小技巧】整数列想保留缺失别用 float 兜底，用可空整数类型：普通 `int64` 列一旦出现缺失就只能升成 float，金额 1200 会变成 1200.0。`astype("Int64")`（大写 I）是 pandas 的可空整数类型，缺失用 `pd.NA` 表示，列还是整数语义；分组求和、排序时它和普通 int 行为一致，写库之前不用再转回 int。

【小技巧】浮点数别用 `==` 比较，用 `np.isclose`：`0.1 + 0.2 == 0.3` 在二进制浮点里是 False，因为 0.1 本身就存不准。判断两列算得“差不多相等”时用 `np.isclose(a, b)`，默认容差是相对 1e-05、绝对 1e-08；需要更严或更松时显式传 `rtol`/`atol`，别靠“保留两位小数再比”蒙过去。

【小技巧】分组后想把聚合结果贴回原表用 `transform`，别用 `merge` 绕一圈：`df.groupby("city")["amount"].transform("sum")` 返回和原表等长的 Series，每行直接拿到它所在分组的合计；`agg` 返回的是分组数那么多行，还得再连回去。算“每行占组内合计的比例”“组内排名”这类需求时，`transform` 一步到位，形状对得上就不用再对齐。

【小技巧】字符串列的批量处理用 `.str` 访问器，别用 `apply` 逐行调字符串方法：`df["city"].str.strip()`、`df["phone"].str.startswith("1")` 都是整列向量化执行，和 `apply(lambda x: x.strip())` 结果一样但快得多。`.str` 后面能接的方法和普通字符串方法基本一一对应；先想“有没有 `.str` 现成方法”，没有再考虑 `apply`。

【小技巧】读 CSV 时把“哪些算缺失”一次告诉 `read_csv`：`pd.read_csv("f.csv", na_values=["", "NA", "N/A", "-"])` 会把这些标记统一当缺失读进来，后面 `isna()` 才能一次数清。默认只认空字符串和 `NaN`，业务里常见的 `-`、`null`、`暂无` 都得显式声明；读进来再逐列替换，不如读入时一次定死。

【小技巧】随机数别再用 `np.random.seed`，用 `np.random.default_rng`：`rng = np.random.default_rng(42)` 返回的 Generator 是 NumPy 1.17 之后推荐的接口，`rng.normal()`、`rng.choice()`、`rng.permutation()` 都挂在它上面。老式全局 `seed` 改的是全局状态，多线程或并行采样时容易互相干扰；每个任务自己建一个 Generator，随机流互不影响，复现时只传种子就行。

【小技巧】`df.append` 已不存在，拼接统一用 `pd.concat`：pandas 2.0 起 `DataFrame.append` 被彻底移除，旧代码里 `df.append(other)` 要改成 `pd.concat([df, other], ignore_index=True)`。顺带记住原文第 6 节的纪律：循环里反复 `concat` 仍是平方级开销，正确姿势永远是先收进 list、最后一次 `concat`。

【小技巧】想知道表到底占多少内存用 `memory_usage(deep=True)`：`df.memory_usage(deep=True).sum()` 会把字符串列的真实内容也算进去，不加 `deep=True` 时对象列只按指针大小估算，数字会严重偏小。排查“读进来就几百 MB”的表时先打这个数，再看是哪列在吃内存；字符串列转 `category` 之前，先用它确认这列到底有多大。

【小技巧】日期列解析失败别让它悄悄变字符串，用 `pd.to_datetime(..., errors="coerce")`：`errors="coerce"` 会把解析不了的日期转成 `NaT`（缺失），后面 `isna()` 能一次数出来；默认 `errors="raise"` 是直接抛错。`read_csv` 里也可以直接用 `parse_dates=["日期列"]` 在读入时解析，日期解析和类型推断一样，越早定死越省事。

【冷知识】Pandas 的名字来自“面板数据”（panel data）：作者 Wes McKinney 2008 年在做金融数据分析时需要处理带时间维度的面板数据，`panel` 一词缩写成了 `pandas`。`DataFrame` 这个名字则借自 R 语言的 `data.frame`，Pandas 早期大量设计都是对着 R 的表格操作抄过来的。

【冷知识】NumPy 是两个早期库合并的产物：2005 年 Travis Oliphant 把 Numeric 和 numarray 合并成 NumPy，`ndarray` 的设计同时继承了两边的接口。NumPy 本身不做表格、不做 SQL，只管“多维数组 + 向量化运算”这一件事；Pandas 的 `Series`/`DataFrame` 底层存的就是 `ndarray`，表格操作最终都落到数组运算上。

【冷知识】`NaN` 不是 Python 或 pandas 发明的，它来自 IEEE 754 浮点标准：`NaN` 是“Not a Number”的缩写，标准规定它和任何值比较都不相等（包括自己）。所以 `NaN != NaN` 是浮点标准的硬性规定，不是 pandas 的 bug；`pd.NA` 是 pandas 后来为“缺失”单独设计的标量，和 `NaN` 不是同一个东西，整数列的可空类型用的就是 `pd.NA` 而不是 `NaN`。

## 8. 面试与自测

1. `reshape` 得到的数组和原数组是什么关系？改一个会影响另一个吗？什么时候必须用 `copy()`？
2. `loc` 和 `iloc` 的区别是什么？什么情况下混用会出错？
3. 为什么 `df[条件]["列"] = 值` 是错的？正确的赋值写法是什么？另外，`NaN` 为什么不能用 `==` 判断？

### 参考答案

1. `reshape` 通常返回视图，只换形状不复制数据，改视图会动原数据；分不清时用 `np.shares_memory` 验证。需要独立数据时显式 `copy()`，比如后续要原地修改又不希望影响原数组的场景。
2. `loc` 按标签/条件取，`iloc` 按位置取。默认整数索引时两者结果一样，但索引被 `set_index` 改过、或不是从 0 开始的连续整数时，`loc[0]`（标签 0）和 `iloc[0]`（第 0 行）就不是一回事，混用会取错行甚至报错。
3. 链式赋值是两次索引：第一次可能返回副本，第二次赋值只改副本，原表不动，还会报 `SettingWithCopyWarning`；正确写法是一次 `loc` 到位（`df.loc[条件, "列"] = 值`）。`NaN` 不等于任何值（包括自己），所以 `== np.nan` 永远是 False，判断缺失只能用 `isna()`。

## 9. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [SQLAlchemy](开发语言/Python/SQLAlchemy.md)：算完表之后持久化到数据库
- [Python性能优化](开发语言/Python/Python性能优化.md)：更大体量的优化手段
