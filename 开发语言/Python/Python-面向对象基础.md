---
知识点: Python 面向对象基础
层级: 基础
适用版本: Python 3.10+
更新日期: 2026-10-31
关联:
  - Python
  - dataclass与typing
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

## 小技巧与冷知识

【小技巧】`__init__` 里别调会被子类重写的方法：`Base.__init__` 里调 `self.setup()`，而 `setup` 在子类里被重写、且要用到子类 `__init__` 后半段才建的属性，调用时属性还不存在，直接报 `AttributeError`。父类初始化只做“父类自己的事”，需要子类配合的步骤改成显式调用（构造完再调），别把“子类还没准备好”藏进父类的构造流程里。

【小技巧】想限制实例属性、省内存时用 `__slots__`：`__slots__ = ("x", "y")` 让实例不再建 `__dict__`，每个实例少一份字典开销，成千上万个小对象时差距明显。代价是实例不能再随意挂新属性（`s.z = 1` 直接报 `AttributeError`），子类想继续省内存也得自己声明 `__slots__`，需要弱引用时还要显式加上 `"__weakref__"`。数据类、配置对象这类“字段固定、数量大”的类最适合，普通业务类没必要上。

【小技巧】只读属性不用写 setter：`@property` 只写 getter 不写 setter 时，赋值直接报 `AttributeError: property has no setter`，这就是现成的只读属性。不用另写“私有变量 + 抛异常的 setter”绕一圈；想表达“这个值只能内部算、外部不能改”，一个没有 setter 的 property 就够了。

【小技巧】`__getattr__` 只在常规查找失败时才被调用：`__getattribute__` 拦截一切属性访问，`__getattr__` 则是“没找到”时的兜底。想做懒加载属性、委托给内部对象时，用 `__getattr__` 最安全——它不会干扰正常属性；反过来，在 `__getattribute__` 里直接写 `self.x` 会无限递归，里面必须用 `object.__getattribute__(self, "x")`。两者分工记死：一个管“找不到”，一个管“所有访问”。

【小技巧】`__eq__` 里对“处理不了的类型”返回 `NotImplemented`，别直接返回 `False`：返回 `NotImplemented` 会让 Python 换对方的 `__eq__` 再试一次（反射比较），两边都不处理才 fallback 成 `False`。直接返回 `False` 会截断这条路，比如自定义的金额类和 `Decimal` 比较时，对方本来能算对，却被你提前判了不等。`NotImplemented` 不是异常，它是“这题我不会，换你来”的返回值；真正表示“不支持这个操作”才抛 `NotImplementedError`。

【小技巧】`super()` 的零参数写法只能写在类体内定义的方法里：在方法里嵌套的 lambda 或内部函数中调 `super()` 会报 `RuntimeError: super(): no arguments`，因为零参数 `super()` 依赖编译器在方法体里注入的 `__class__` 单元格，嵌套函数拿不到它。嵌套函数里需要调父类方法时，写全参数版 `super(类名, self)`；或者把逻辑抽成独立方法，别在闭包里偷用 `super()`。

【小技巧】浅拷贝只拷一层，嵌套的可变属性还是同一份：`copy.copy(obj)` 复制的是对象本身，`obj.items` 这个列表仍是原对象那个，改副本的列表、原对象也跟着变。想彻底断开用 `copy.deepcopy`；只想断开某一层时，拷贝后手动把那一层换成新对象。需要定制拷贝行为时可以实现 `__copy__` / `__deepcopy__`，比如含文件句柄、锁的对象不该被深拷贝，里面显式决定“哪些字段共享、哪些重建”。

【小技巧】不要靠 `__del__` 做资源清理：`__del__` 什么时候跑、跑不跑，取决于 GC 和引用关系，循环引用、解释器退出时都可能不按预期触发。需要确定性清理时用上下文管理器（`with` 语句 + `__enter__` / `__exit__`），或者 `weakref.finalize` 注册一个“对象被回收时执行”的回调——前者管“用完就关”，后者管“没人用了再收尾”，都比 `__del__` 靠谱。

【小技巧】想让类名本身可调用出不同实例，用类方法做替代构造器：`@classmethod def from_dict(cls, d): return cls(**d)`，`cls` 会自动是调用它的那个类，子类调用时返回的就是子类实例；写成 `@staticmethod` 再硬编码类名，子类调用时拿到的还是父类实例。`__init__` 保持只收“最核心的参数”，多种来源的构造（从字典、从 JSON、从默认值）各写一个类方法，构造逻辑不挤在 `__init__` 里。

【小技巧】`dataclass` 需要“构造后算派生字段、做校验”时用 `__post_init__`：`@dataclass` 生成的 `__init__` 跑完后会自动调 `__post_init__`，在里面可以由已有字段算出派生值、或校验字段组合是否合法。但注意 `frozen=True` 的类在 `__post_init__` 里也不能直接赋值，要用 `object.__setattr__(self, "x", ...)` 绕过冻结；派生字段本身用 `field(init=False)` 声明，明确告诉读者“这个字段不由构造参数给”。

【小技巧】插件注册表用 `__init_subclass__` 实现，别靠 import 时手动登记：父类里定义 `def __init_subclass__(cls, **kwargs)`，每个子类定义完成时它自动执行一次，在里面把子类按名字收进字典。比“每个子类文件底部写一行 `register(Xxx)`”可靠——后者漏写一行就静默丢一个插件；比元类轻得多，`__init_subclass__` 就是专门为“子类出生时干点事”留的钩子。

【小技巧】抽象方法也可以有实现：`@abstractmethod` 的方法体不是摆设，子类重写后仍能用 `super().send(...)` 调到父类的那份实现。适合“所有子类都要先做同一段公共逻辑、再做自己的事”的场景——把公共部分写进抽象方法体里，子类先 `super()` 再补自己的，公共逻辑只写一次；抽象方法的存在只保证“子类必须重写”，不保证“父类那份没人用”。

【小技巧】判断类型用 `isinstance`，别用 `type(x) is Y`：`type(x) is Account` 会把 `SavingsAccount` 的实例判成“不是”，而 `isinstance` 会认子类，符合多态的预期。反过来，需要“精确就是这个类、子类也不算”时才用 `type(x) is Y`，那是刻意收紧；默认选 `isinstance`，除非你有明确理由要排除子类。

【小技巧】`getattr(obj, name, default)` 的第三个参数是现成的兜底：`getattr(cfg, "timeout", 30)` 一行解决“有就用、没有就用默认值”，不用先 `hasattr` 再 `getattr` 查两遍。注意默认值只在属性确实不存在时返回，属性存在但值是 `None` 时返回的是 `None`——“不存在”和“值是 None”是两回事，兜底只管前者。

【冷知识】类本身也是对象：`type(Account)` 是 `type`，`Account` 是 `type` 的实例，所以类可以像普通对象一样被赋值、传参、运行时挂属性。`type("K", (Base,), {"v": 42})` 这种三参数形式就是在运行期动态造一个类；元类（metaclass）就是“造类的类”，`class X(metaclass=M)` 里的 `M` 本质上是重写了“类是怎么被造出来的”。

【冷知识】`bool` 是 `int` 的子类：`isinstance(True, int)` 是 `True`，`True + True` 等于 `2`。这不是巧合，Python 里布尔值在历史上就是用 0 和 1 实现的，`True`/`False` 是 `int` 的两个单例。所以 `isinstance(x, int)` 会把布尔值也算进去，需要排除时要显式先判 `isinstance(x, bool)`；`True == 1` 为真也是同一根子。

【冷知识】`self` 只是个约定，不是关键字：第一个参数叫什么都行，写 `this` 也能跑，解释器只认位置不认名字。之所以全世界都写 `self`，是因为 PEP 8 和所有人的习惯；显式写出来而不是像 Java 一样隐式注入，是 Python“显式优于隐式”的一个缩影——方法调用 `obj.m()` 时解释器只是把 `obj` 当第一个实参传了进去。

【冷知识】`__init__` 不是构造方法，`__new__` 才是：实例的内存和对象本身由 `__new__` 创建，`__init__` 只负责初始化；所以 `__init__` 返回非 `None` 会直接报 `TypeError`，而 `__new__` 必须返回实例。平时改 `__init__` 就够了，只有继承不可变类型（`int`、`str`、`tuple`）时才需要重写 `__new__`，因为不可变对象的值在创建那一刻就定死了，`__init__` 里再改已经晚了。

【冷知识】`object` 是所有类的祖先，连 `type` 也不例外：`isinstance(type, object)` 是 `True`，同时 `isinstance(object, type)` 也是 `True`——“类是 type 的实例、type 是 object 的子类、object 又是 type 的实例”，这组循环关系是 Python 类型系统的根。`dir(object)` 里只有最基础的 `__init__`、`__str__`、`__eq__` 等方法，每个类都是从这套最小接口出发往上叠的。

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
