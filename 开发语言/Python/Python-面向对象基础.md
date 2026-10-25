---
知识点: Python 面向对象基础
层级: 基础
适用版本: Python 3.10+
更新日期: 2026-10-25
关联:
  - Python
  - dataclass 与 typing
  - 装饰器
标签:
  - Python
  - 实践
---

# Python 面向对象基础

## 1. 核心概念：类与对象

类是对象的模板，对象是类的实例。`__init__` 是初始化方法，不是构造方法：实例已由 `__new__` 创建出来，`__init__` 只负责填初始状态，且不能有返回值。

```python
class Account:
    bank = "ICBC"  # 类属性：所有实例共享

    def __init__(self, owner: str, balance: float = 0.0):
        self.owner = owner      # 实例属性：每个实例一份
        self.balance = balance

    def deposit(self, amount: float) -> None:
        self.balance += amount
```

三条分界线要先立住：

- **实例属性 vs 类属性**：`self.x` 是实例的，`ClassName.x` 是全类共享的。实例读属性时先查实例、再查类；实例赋值永远只写实例，不会改到类属性。
- **可变类属性是经典坑**：类属性是 `[]` 或 `{}` 时，所有实例改的是同一份。默认状态一律放 `__init__` 里建，别放类属性上。
- **三种方法**：实例方法（第一个参数 `self`）、类方法（`@classmethod`，第一个参数 `cls`，常用于替代构造器）、静态方法（`@staticmethod`，不拿 `self` 也不拿 `cls`，只是挂在类名下的普通函数）。

## 2. 继承与多态

子类继承父类的属性和方法，并可以重写。`super()` 不是“调父类”，而是“按 MRO 找下一个”——单继承下它就是父类，多继承下它走的是方法解析顺序（MRO，C3 线性化）。

```python
class SavingsAccount(Account):
    def __init__(self, owner: str, balance: float = 0.0, rate: float = 0.02):
        super().__init__(owner, balance)  # 先让父类把通用状态建好
        self.rate = rate

    def deposit(self, amount: float) -> None:
        super().deposit(amount)  # 复用父类逻辑，再补自己的
        self.balance += amount * self.rate
```

- **MRO 怎么查**：`SavingsAccount.__mro__` 直接打印顺序。菱形继承（D 同时继承 B、C，B、C 都继承 A）时，MRO 保证 A 的方法只走一次，且顺序可预测；`super()` 链上的每一环都要调 `super()`，断一环后面就全断。
- **多态靠协议不靠继承**：Python 里“能调什么方法”比“是什么类型”重要。函数接受任何带 `deposit` 方法的对象就能跑，这叫鸭子类型——走起来像鸭子，就按鸭子用。

## 3. 魔术方法

以双下划线包围的方法由解释器在特定语法点自动调用，常用的就这批：

| 方法 | 触发点 | 注意 |
|---|---|---|
| `__init__` | 实例创建后初始化 | 不能 return 值 |
| `__str__` | `str(obj)`、print | 给人看的 |
| `__repr__` | `repr(obj)`、交互式回显 | 给开发者看的，能还原对象最好 |
| `__eq__` | `==` 比较 | 只定义 `__eq__` 不定义 `__hash__`，`__hash__` 会被置为 `None`，对象立刻不能当 dict key、不能进 set |
| `__len__` | `len(obj)` | 必须返回 `>= 0` 的整数 |
| `__getitem__` | `obj[key]`、切片、for 迭代兜底 | 定义了它，没定义 `__iter__` 时 for 也能跑 |
| `__iter__` | `for`、`unpack` | 应返回迭代器 |
| `__call__` | `obj()` | 让实例像函数一样被调用 |

只实现 `__str__` 不实现 `__repr__` 时，调试打印默认回退到 `__repr__`，所以两者至少要实现一个；规范做法是 `__repr__` 写全量信息、`__str__` 写给人看的摘要。

## 4. 属性、描述符与 dataclass

### 4.1 @property：把方法调用伪装成属性访问

```python
class Account:
    def __init__(self, balance: float = 0.0):
        self._balance = balance

    @property
    def balance(self) -> float:
        return self._balance

    @balance.setter
    def balance(self, value: float) -> None:
        if value < 0:
            raise ValueError("余额不能为负")
        self._balance = value
```

调用方写 `acc.balance = -5` 仍会触发校验，因为 `balance` 是数据描述符。注意 `__init__` 里也应走 `self.balance = ...` 而不是直接写 `self._balance`，否则校验会被绕过。

### 4.2 描述符入门

描述符就是实现了 `__get__`、`__set__`、`__delete__` 中任意方法的类。`@property` 本质上就是一个描述符：访问实例属性时，解释器先看类的属性里有没有数据描述符，有就走描述符逻辑，没有才走实例字典。理解这一条，就理解了 property、类方法、静态方法的统一机制。日常写业务代码用 `@property` 就够了，自己写描述符只在做 ORM 字段、类型校验框架这类复用组件时才上。

### 4.3 dataclass 入门

字段多、只做数据承载的类别手写 `__init__`/`__repr__`/`__eq__`，用 `@dataclass`：

```python
from dataclasses import dataclass, field

@dataclass
class LoanApplication:
    applicant: str
    amount: float
    tags: list = field(default_factory=list)  # 可变默认值必须用 field 包起来
```

可变默认值（`tags: list = []`）是 dataclass 的头号坑：所有实例会共享同一份列表，必须用 `field(default_factory=list)`。`frozen=True` 可生成不可变版本。完整用法（字段选项、排序、继承规则）见 [dataclass 与 typing](开发语言/Python/dataclass与typing.md)，本篇不再展开。

## 5. 抽象基类与鸭子类型

```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class SmsNotifier(Notifier):
    def send(self, message: str) -> None:
        ...
```

- **ABC**：含抽象方法的类不能实例化，子类不实现全部抽象方法就还是抽象类。适合“必须实现某组方法”的硬约束，如插件接口、驱动层。
- **鸭子类型**：不做任何声明，只要对象有对应方法就能用。适合内部函数、工具函数、小范围协作，灵活但靠约定。
- **Protocol（3.8+，typing）**：介于两者之间——结构化类型：声明“有这些方法就算符合”，不要求继承，静态检查工具能校验。公开 API 想给类型提示又不想强迫继承时用它。

选型口诀：内部小函数用鸭子类型，对外接口、插件体系用 ABC 或 Protocol，别为了“面向对象”给每个小类都套继承。

## 6. 常见误区

- 误区：类属性就是“默认值”，实例改了不影响别人。可变类属性全实例共享，改了就是改了所有人。
- 误区：`__init__` 里写 `self.balance` 和 `self._balance` 没区别。走 property 会触发校验、直接写内部变量会绕过。
- 误区：多继承里 `super()` 等于父类。`super()` 走 MRO，菱形继承下它可能指向兄弟分支，不是直系父类。
- 误区：定义了 `__eq__` 对象还能当 key。没配 `__hash__` 时哈希会被置空，`set`/`dict` 直接报错。
- 误区：继承层数越深越面向对象。超过两三层的继承通常意味着抽象错了，组合（把对象当属性）往往更稳。

## 7. 面试与自测

1. 类属性和实例属性的区别？可变类属性被所有实例共享会出什么问题？
2. `super()` 在多继承里按什么顺序找方法？`__mro__` 怎么看？
3. 只定义 `__eq__` 不定义 `__hash__`，对象还能放进 set、当 dict key 吗？为什么？
4. `@property` 的 setter 是怎么回事？为什么说 property 本质是描述符？
5. 鸭子类型和 ABC 各适合什么场景？什么时候该用 Protocol？

### 参考答案

1. 实例属性每个实例一份，存在实例字典里；类属性全类共享，存在类字典里，实例读属性时先查实例再查类。类属性是 `[]`、`{}` 这类可变对象时，所有实例改的是同一份，一个实例 append，所有实例都看得到；默认状态应放 `__init__` 里建。
2. `super()` 不等于“调父类”，而是按 MRO（C3 线性化）找当前类之后的下一个类。`ClassName.__mro__` 直接打印顺序；菱形继承下 MRO 保证公共基类的方法只走一次，且 `super()` 链上每一环都要调 `super()`，断一环后面全断。
3. 不能。只定义 `__eq__` 时 Python 会把 `__hash__` 置为 `None`，对象变成不可哈希，`set` 添加或当 dict key 会直接报错。规则是：自定义相等语义就要同时给出哈希语义，要么显式定义 `__hash__`，要么接受不可哈希。
4. `@balance.setter` 装饰的方法在 `acc.balance = value` 时被调用，可在里面做校验（如余额不能为负）。property 本质是描述符：它实现了 `__get__`/`__set__`，实例属性访问时解释器优先走类属性里的描述符逻辑，所以“赋值”实际走的是 setter 方法。
5. 内部小函数、工具函数用鸭子类型：不声明类型，只要有方法就能用，灵活。插件接口、驱动层这类“必须实现某组方法”的硬约束用 ABC（不实现就不能实例化）。公开 API 想给静态检查又不想强迫继承时用 Protocol：声明方法结构，符合结构就算符合，不要求继承。

## 8. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与常见坑总览。
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：dataclass 字段选项、排序、继承规则的完整用法。
- [装饰器](开发语言/Python/装饰器.md)：`@property`、`@classmethod` 本身就是装饰器，原理与手写装饰器见本篇。
- [Python 多进程与 GIL](开发语言/Python/Python-多进程与GIL.md)：类与对象要跨进程传时需可 pickle，相关约束见本篇。
