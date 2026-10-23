---
知识点: SQLAlchemy
层级: 进阶
适用版本: SQLAlchemy 2.x
更新日期: 2026-10-23
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

## 9. 关联知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [pydantic 数据校验](开发语言/Python/pydantic.md)：落库前校验形状，SQLAlchemy 管落库
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：服务层出口转 dataclass 传递
- [MySQL-数据库](数据库/MySQL-数据库.md)：SQL 与事务基础，换引擎不换原理

## 10. 一句话总结

SQLAlchemy = 一半 MyBatis 的可控 + 一半 Hibernate 的托管：Core 管得住 SQL，ORM 省得掉样板，Session 是两者共同的事务边界。
