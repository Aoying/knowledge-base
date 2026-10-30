---
知识点: SQLAlchemy
层级: 进阶
适用版本: SQLAlchemy 2.x
更新日期: 2026-10-31
关联:
  - Python
  - pydantic
  - MySQL-数据库
标签:
  - Python
  - 数据库
  - 实践
---

# SQLAlchemy

## 1. 核心概念

SQLAlchemy 是 Python 的 SQL 工具包，分两层：

- **Core**：SQL 表达式语言 + 连接池 + 方言层，用 Python 对象拼 SQL，接近手写 SQL 的思维。
- **ORM**：在 Core 之上把表映射成类、行映射成对象，带 Session 做状态跟踪、事务与关系加载。

定位：小说引擎的世界状态库（SQLite）全靠它——Entity、Relation、Timeline、Event、Snapshot、Chapter 等模型都是 ORM 映射，分析结果经它落库，ChapterContextService 经它聚合上下文。选它的原因正是两层可混用：常规增删改查走 ORM，复杂聚合（时间线窗口、状态回溯）下沉到 Core 表达式。

口径提醒：本文按 **2.0 风格**写（`select()` + `scalars`、`Mapped[]` 映射）。1.x 的 `session.query()` 在 2.x 仍兼容但已不推荐；版本细节以官方文档为准。

## 2. 基础知识：映射与 Session

```python
from sqlalchemy import ForeignKey, String, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, sessionmaker

class Base(DeclarativeBase):
    pass

class Entity(Base):
    __tablename__ = "entity"
    id: Mapped[int] = mapped_column(primary_key=True)
    project_id: Mapped[int] = mapped_column(index=True)
    name: Mapped[str] = mapped_column(String(64))
    type: Mapped[str] = mapped_column(String(32))      # 人物/地点/物品/组织
    states: Mapped[list["EntityState"]] = relationship(back_populates="entity")

class EntityState(Base):
    __tablename__ = "entity_state"
    id: Mapped[int] = mapped_column(primary_key=True)
    entity_id: Mapped[int] = mapped_column(ForeignKey("entity.id"))
    chapter: Mapped[int]
    field: Mapped[str] = mapped_column(String(64))     # location / alive / mood
    value: Mapped[str] = mapped_column(String(256))
    entity: Mapped[Entity] = relationship(back_populates="states")

engine = create_engine("sqlite:///novel.db")
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
```

- **engine 全局只建一个**：它内部管连接池；每处业务 `create_engine` 等于每处一套池，纯属浪费。
- **Session 是工作单元（Unit of Work）**：一个业务动作一个 Session（如"写一章的分析结果"），用完即关。标准写法 `with SessionLocal() as session, session.begin():`——begin 块内自动提交、异常自动回滚，出 with 连接归还池。
- `expire_on_commit=False` 值得显式设置：默认提交后对象属性过期，下次访问触发懒加载刷新，Session 已关闭时会抛 `DetachedInstanceError`。长流程批处理关掉过期更省心，代价是对象可能是旧快照。

## 3. 查询：2.0 风格

```python
from sqlalchemy import select

with SessionLocal() as session:
    charas = session.scalars(   # 查实体列表用 scalars，不要 execute 的 Row
        select(Entity).where(Entity.project_id == pid, Entity.type == "人物")
    ).all()
    # 取某人物第 N 章前最近一条状态：排序 + limit，Core 表达式下沉
    state = session.scalar(
        select(EntityState)
        .where(EntityState.entity_id == eid, EntityState.chapter <= n)
        .order_by(EntityState.chapter.desc()).limit(1)
    )
```

条件走 `==`、`in_()`、`.like()` 等表达式 API 组合，**不要用 f-string 拼 SQL**（防注入见第 6 节）。

## 4. 关系加载策略与 N+1

`relationship` 默认懒加载：访问 `entity.states` 时才单独发一条查询。遍历 50 个人物各取状态就是 1 + 50 条 SQL——经典 N+1。批量场景按需改预加载，如 `.options(selectinload(Entity.states))` 用一条额外 IN 查询把集合批量带出：

| 策略 | 机制 | 适用 |
|---|---|---|
| `select`（默认懒加载） | 访问时单查 | 单对象、访问面不确定 |
| `selectinload` | 主查询后一条 IN 批量查 | 一对多/多对多集合，最常用 |
| `joinedload` | JOIN 一次带出 | 多对一/一对一；一对多会行膨胀，别滥用 |
排查手段：开 `echo=True` 数 SQL 条数，N+1 是一排同构 SQL；`raiseload` 策略可让懒加载访问直接报错，排查期好用。ChapterContextService 这类聚合入口默认就应显式 `selectinload`，不要靠懒加载逐个触发。

## 5. 事务与并发口径

- **flush 与 commit 分清**：`flush` 把变更发到数据库（拿到自增主键、触发约束检查）但未提交；`commit` 才落定。先拿 id 再建关联时手动 `flush()`，别为了拿 id 提前 commit 把业务动作拆成两个事务。
- **SQLite 的写并发**：SQLite 单写模型，同一时刻只有一个写事务。引擎是单机批处理，串行写够用；出现 `database is locked` 先查有没有 Session 漏关（尤其异常路径）。用 `event` 监听 connect 事件执行 `PRAGMA journal_mode=WAL` 开 WAL，读写互不阻塞，低成本缓解。

- 分析回写**整章一个事务**：一章的 entities/relations/events 要么全落，要么全回滚重抽；落一半的世界状态比不落更糟（矛盾检测会把半成品当事实）。

## 6. 防注入：参数化是默认行为

用表达式 API 或 ORM 查询时，输入都是绑定参数传给驱动，天然参数化。危险的是绕开它：

```python
# ❌ LLM 抽出的 name 直接拼 SQL，单引号就能打穿
session.execute(text(f"SELECT * FROM entity WHERE name = '{name}'"))
# ✅ text() 也必须用绑定参数
session.execute(text("SELECT * FROM entity WHERE name = :name"), {"name": name})
```

引擎的输入是 LLM 输出，本质不可信（见 pydantic 篇）：校验管形状，参数化管注入，两道防线各管一段。

## 7. 与 Java MyBatis / Hibernate 的思维对照

| 维度 | MyBatis | Hibernate / JPA | SQLAlchemy |
|---|---|---|---|
| 抽象层级 | SQL 映射，手写 SQL 为主 | 全自动 ORM | Core + ORM 双层可混 |
| 会话概念 | SqlSession，无状态执行器 | EntityManager，托管状态 | Session，带 identity map 与 Unit of Work，最接近 Hibernate |
| 对象跟踪 | 无，查出来即 POJO | 有，脏检查自动 flush | 有，脏检查自动 flush（与 MyBatis 差别最大） |
| 关系加载 | 手写 resultMap / 嵌套查询 | 注解 + fetch 策略 | relationship + 加载策略，N+1 三家同款 |

迁移提示：从 MyBatis 过来最容易踩的是**把 Session 当无状态执行器**——改了托管对象属性不调任何方法，commit 时自己就 flush 出 UPDATE（脏检查）。是特性不是 bug，但"顺手赋值试试"的调试习惯会直接改库。

## 8. 常见误区

- **循环里逐个 commit**：性能差且破坏原子性；批量落库一次事务收口。
- **ORM 对象当 DTO 跨层乱传**：出了 Session 作用域再访问懒加载属性就炸；服务层出口转 dict / dataclass 再传。
- **cascade 删除图省事**：状态历史、快照大多只追加不删除，cascade 误配会删掉一片历史且不可恢复；删除策略按表逐个定，默认不配。

## 小技巧与冷知识

【小技巧】按主键取一行用 `session.get(Entity, id)`，别写 `select(...).where(id == ...)`：`get` 会先查 Session 的 identity map，对象已在内存里就直接返回、不发 SQL；查不到才发一条主键查询。循环里反复取同一批实体时，这个“先看内存再查库”的顺序能省掉大量重复 SQL；按非主键字段查才用 `select`。

【小技巧】取“有且仅有一个”时用 `scalar_one_or_none()`，别用 `scalars().first()` 糊弄过去：`first()` 在有多个结果时只取第一个、悄悄吞掉“本该唯一却重复了”的数据问题；`scalar_one_or_none()` 在多于一行时直接抛 `MultipleResultsFound`，等于把唯一性约束也写进了查询里。查“某个项目下同名实体”这类本该唯一的场景，先想好“多了一行该不该炸”。

【小技巧】集合上用 `joinedload` 时，结果要调 `.unique()`：JOIN 会把一对多集合的行数“膨胀”成笛卡尔积，同一实体会出现多次，`session.scalars(...).unique().all()` 才能按实体去重。忘了 `.unique()`，返回的列表里同一个 Entity 会出现好几次，计数、去重逻辑全被带偏；`selectinload` 没有这个问题，因为它用的是第二条 IN 查询。

【小技巧】批量插数据用 Core 的 `insert()` 一次传多行，别循环 `session.add()`：`session.execute(insert(Entity), [{...}, {...}])` 会走驱动的 executemany，一次发多行；循环里逐个 `add` 再一次性 `commit` 虽然也行，但每行都要经过 ORM 的状态跟踪与属性处理。批量落库（比如一章解析出的几十条状态）先走 Core 批量插入，单条、有业务逻辑的写操作再走 ORM。

【小技巧】批量改、批量删也别先查出来再逐个改：`session.execute(update(Entity).where(...).values(field="x"))`、`session.execute(delete(EntityState).where(...))` 直接在数据库层面改，不加载对象、不触发 ORM 的逐行处理。注意改完后 Session 里的同类对象可能还是旧快照，顺手调一次 `session.expire_all()` 让它们下次访问时重新加载，否则“改完了、读出来还是旧的”。

【小技巧】`autoflush` 默认是开的，查询前会先把挂起的变更发出去：`session.add(obj)` 后还没 commit 就查 `select(Entity)`，Session 会先自动 `flush` 再查，所以查得到刚加的对象。这条默认很方便，但批量构造对象中途查一次库，会提前触发约束检查、甚至把半成品写进数据库；批量组装阶段可以 `SessionLocal(autoflush=False)`，攒够了再手动 `flush()`。

【小技巧】调试“这次 commit 到底要写什么”看 `session.new` / `session.dirty` / `session.deleted`：这三个集合分别装着待插入、待更新、待删除的对象，`flush` 前打出来就知道这次事务要动哪些行。对象状态再细一点用 `sqlalchemy.inspect(obj)`，`pending`、`persistent`、`deleted`、`detached` 一目了然；怀疑“改了属性怎么没生成 UPDATE”时，先看对象在不在 `dirty` 里。

【小技巧】SQLite 的外键默认是关着的：SQLite 本身默认不执行外键约束，SQLAlchemy 不会替你打开。建引擎后用事件监听在每次连接时执行 `PRAGMA foreign_keys=ON`，否则 `ForeignKey("entity.id")` 只在建表时留个声明，删了父行、子行还在，级联、约束全是摆设。换 MySQL/PostgreSQL 不用管这条，只用 SQLite 时必须补上。

【小技巧】多线程共用 SQLite 引擎时要传 `check_same_thread=False`：Python 的 sqlite3 驱动默认禁止跨线程用同一个连接，不传这个参数，多线程里复用连接池的连接会直接抛 `ProgrammingError`。`create_engine("sqlite:///novel.db", connect_args={"check_same_thread": False})`，再配 `poolclass=NullPool` 让每次用完就关连接，别让连接在线程间流转；单线程批处理不用管这条。

【小技巧】计数别把全表查出来再 `len()`：`len(session.scalars(select(Entity)).all())` 是把所有行加载成对象再数，表一大就爆内存；`session.scalar(select(func.count()).select_from(Entity))` 只返回一个数字。凡是“只要个数、不要行”的场景，一律让数据库算，别把数据拉回 Python 再数。

【小技巧】MySQL 这类远程库加 `pool_pre_ping=True`：连接在池里闲置太久，服务端可能已悄悄断开，下次拿出来直接用会报 `MySQL server has gone away`。`pool_pre_ping` 让每次取连接前先发一个轻量 ping 探活，断了就重建；代价是一次往返，换来的是“闲置后第一次请求不炸”。SQLite 是本地文件，不用开这个。

【小技巧】想给所有表自动填 `updated_at`，用 `before_flush` 事件，别在业务代码里逐个赋值：`@event.listens_for(Session, "before_flush")` 里遍历 `session.dirty`，有 `updated_at` 字段的统一赋当前时间。时间戳、版本号这类“每行都要、业务不关心”的字段，集中到事件里处理，业务代码里就不会出现“这个表忘了、那个表写了”的遗漏。

【小技巧】约束和索引的名字要先定好命名约定，否则迁移工具认不出它们：`MetaData(naming_convention={"ix": "ix_%(column_0_label)s", "uq": "uq_%(table_name)s_%(column_0_name)s", "fk": "fk_%(table_name)s_%(column_0_name)s"})` 这类约定让 Alembic 自动生成迁移时能按名字识别已有的约束；没约定，自动生成的迁移会把“改名”当成“删了重建”，生产库执行一次就多一次风险。

【小技巧】`text()` 写原生 SQL 时，返回值想按列名取，先调 `.columns()`：`session.execute(text("SELECT id, name FROM entity").columns(id=Integer, name=String))` 声明每列的类型，之后才能用 `row.name` 或 `row._mapping["name"]` 取值；不声明，返回的 Row 只能按位置下标取，SQL 一改顺序，取值的代码全错位。原生 SQL 只在“Core 表达式写不出来”时用，用了就要把类型声明补齐。

【冷知识】SQLAlchemy 的作者是 Michael Bayer，2006 年发布第一个版本：它最初就是“SQL 工具包 + 方言层”的 Core，ORM 是后来才长出来的第二层，这也解释了为什么它和 Hibernate 的路线不同——Hibernate 生来就是 ORM，SQLAlchemy 是先有了拼 SQL 的工具，再在上面盖了对象映射。

【冷知识】SQLAlchemy 2.0 发布于 2023 年 1 月，是它十多年来最大的一次风格切换：`session.query()` 被正式列为 legacy，`select()` + `scalars()` 成了唯一推荐写法；同时 `Mapped[]` 注解映射取代了 `Column` 声明式写法。老教程里满屏的 `query().filter().all()`，在 2.0 风格下全部要换写法，读旧代码时先认版本再认 API。

【冷知识】Alembic 和 SQLAlchemy 是同一个作者：Michael Bayer 写完 SQLAlchemy 之后，又写了 Alembic 做数据库迁移，两者配合的约定（比如上面那条 naming_convention）就是他自己留的接口。Python 生态里“ORM + 迁移”这对组合出自同一人之手，迁移工具能读懂 ORM 的模型定义不是巧合，是有意设计。

【冷知识】`session` 里的 identity map 是“同一行在同一个 Session 里永远是同一个对象”：两次查到同一行，返回的是同一个 Python 对象，改一处、另一处立刻跟着变。这正是第 7 节说的“脏检查”的底座——Session 靠 identity map 知道“这个对象就是数据库里的那一行”，改了属性才知道要生成 UPDATE；出了 Session，这个保证就不存在了。

## 面试与自测

1. 遍历 50 个人物各取 states 为什么会变成 1 + 50 条 SQL？怎么改？
2. flush 和 commit 有什么区别？什么时候需要手动 flush？
3. Session 已关闭后，再访问提交过的对象属性为什么会抛 DetachedInstanceError？怎么避？

### 参考答案

1. 因为 relationship 默认懒加载：访问 entity.states 时才单独发一条查询，遍历 50 个人物就是 1 + 50 条，这就是经典 N+1。批量场景改预加载，如 .options(selectinload(Entity.states))，用一条额外 IN 查询把集合批量带出；joinedload 是 JOIN 一次带出，适合多对一/一对一，一对多会行膨胀，别滥用。排查手段是开 echo=True 数 SQL 条数，N+1 是一排同构 SQL；raiseload 可让懒加载访问直接报错，排查期好用。
2. flush 只是把变更发到数据库（拿到自增主键、触发约束检查）但未提交，commit 才落定。先拿 id 再建关联时手动 flush()，别为了拿 id 提前 commit 把业务动作拆成两个事务；批量落库也要一次事务收口，不能循环里逐个 commit。
3. 因为默认 expire_on_commit=True，提交后对象属性过期，下次访问触发懒加载刷新，而 Session 已关闭就抛 DetachedInstanceError。避法是显式设置 expire_on_commit=False（代价是对象可能是旧快照），或按第 8 节的做法：服务层出口转 dict / dataclass 再传，不要把 ORM 对象当 DTO 跨层乱传。

## 9. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [pydantic 数据校验](开发语言/Python/pydantic.md)：落库前校验形状，SQLAlchemy 管落库
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：服务层出口转 dataclass 传递
- [MySQL-数据库](数据库/MySQL/MySQL-数据库.md)：SQL 与事务基础，换引擎不换原理

## 10. 一句话总结

SQLAlchemy = 一半 MyBatis 的可控 + 一半 Hibernate 的托管：Core 管得住 SQL，ORM 省得掉样板，Session 是两者共同的事务边界。
