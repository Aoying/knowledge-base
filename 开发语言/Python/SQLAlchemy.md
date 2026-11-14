---
知识点: SQLAlchemy
层级: 进阶
适用版本: SQLAlchemy 2.x
更新日期: 2026-11-14
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

## 批量与连接池定量：N+1、批量落库与连接池算一算

口径先说清：本节只算条数、倍数与等待时间，不重复第 4 节的加载策略定性结论与第 5 节的事务口径。所有耗时数字均为估算——单条 SQL 的往返时延、每次 commit 的 fsync 成本因磁盘、网络与后端而异，生产值以实测为准；条数、倍数与连接数是确定的换算，可以直接拿去对账。

### N+1：先数条数，再谈策略

懒加载的条数公式是 `1 + N`：1 条主查询，N 个父对象各触发 1 条。预加载把 N 压成常数：`selectinload` 是 1 条主查询 + 每个集合 1 条 IN 查询；`joinedload` 是 1 条，但行数会膨胀。

【算例 1】聚合 50 个人物及各自的状态，每人平均 4 条状态：

- 懒加载：1 + 50 = 51 条 SQL。
- `selectinload(Entity.states)`：1 条主查询 + 1 条 `WHERE id IN (...)` = 2 条，省下 49 条，约为原来的 1/25（51 ÷ 2 ≈ 25.5 倍）。
- `joinedload`：1 条 SQL，但返回 50 × 4 = 200 行，行数是父行数的 4 倍——父对象列被重复了 4 遍，内存与传输量跟着涨 4 倍，还要 `.unique()` 去重。
- 再加一个集合（如 relations，每人平均 3 条）：懒加载变成 1 + 50 + 50 = 101 条；`selectinload` 两个集合是 1 + 2 = 3 条；`joinedload` 两层集合是 50 × 4 × 3 = 600 行，膨胀 12 倍。

换算口诀：集合数每加 1，`selectinload` 只加 1 条；`joinedload` 的行数按每层平均子行数连乘。ChapterContextService 这类一次聚合多个集合的入口，条数预算直接按 `1 + 集合数` 算。

### 批量落库：提交次数才是大头

逐条 `add()` + 逐条 `commit()` 的成本不在 INSERT 本身，而在提交次数：N 行就是 N 次 commit，每次 commit 至少一次往返 + 一次 fsync。一次事务收口（第 5 节口径）把提交次数压到 1 次；再走 Core `insert()` 的 executemany，还能把 N 次 INSERT 调用合并成少量往返。

【算例 2】一章解析出 1000 条 EntityState，假设每次 commit 的 fsync 约 5 毫秒（本地 SSD 估算值）：

- 逐条 commit：1000 次提交 × 5 毫秒 = 5000 毫秒 ≈ 5 秒，只算提交，不含 INSERT 本身。
- 一次事务 + Core 批量 insert：1 次提交 × 5 毫秒 = 5 毫秒，提交成本降为 1/1000。
- 需要主键回填时，2.0 的 `insertmanyvalues` 默认每页 1000 行：1000 行 = 1 页、1 次往返取回主键；10000 行 = 10 页、10 次往返（`ceil(10000 ÷ 1000) = 10`）。

【算例 3】flush 频率：`flush` 只发 SQL、不提交，成本是 SQL 往返而非 fsync。10000 行每 1000 行 `flush()` 一次是 10 次 flush；攒满再 flush 比每行 flush 少 9990 次往返。注意 `autoflush` 默认开启，批量组装中途查一次库就会提前触发 flush（见小技巧节），大批量组装时先关掉再手动 flush。

调用次数也要换算：逐条 `add()` + `flush()` 是 1000 次 execute 调用，每次都要走一次 ORM 的状态跟踪与 SQL 编译；Core `insert()` 配 executemany 是一次调用带 1000 行参数，驱动层面分批发出，Python 侧的调用次数从 1000 降到 1。10000 行同理：逐条是 10000 次调用，executemany 是 1 次调用（驱动内部再按批次发）。

批量路径的代价要记清：`bulk_save_objects` 与 Core insert 跳过 ORM 的状态跟踪与部分事件，对象不进 identity map；`bulk_save_objects(return_defaults=True)` 会逐行取回主键，性能大跌，需要主键时优先用带 RETURNING 的 `insertmanyvalues`（SQLite ≥ 3.35 支持 RETURNING）。

### 连接池：最大连接数是一道算术题

QueuePool 的默认值（官方文档）：`pool_size = 5`，`max_overflow = 10`，`pool_timeout = 30` 秒。最大并发连接数 = 5 + 10 = 15；第 16 个并发请求起排队等连接，超过 30 秒拿不到就报超时。溢出连接用完即断开丢弃，只有 `pool_size` 的 5 个会常驻池中。

【算例 4】20 个并发请求同时到达，每请求持有连接 200 毫秒：

- 前 15 个立即拿到连接，剩下 5 个排队；按每批 15 个、每批 200 毫秒算，2 批约 0.4 秒消化完，无人超时。
- 同样 20 个并发，若每请求持有连接 5 秒：第 16–20 个要等约 5 秒才能拿到连接（仍在 30 秒超时内）；持有时间涨到 30 秒以上（如 40 秒）才会有人排队超 30 秒报超时——排队等待约等于一个持有时长。
- 池够不够用 Little 定律估：并发连接数 ≈ 到达率 × 平均持有时间。50 请求/秒 × 0.2 秒 = 10 个连接，15 的上限够用；50 请求/秒 × 0.5 秒 = 25 个，超了 15，要么调大池，要么缩短持有时间（Session 用完即关，别让连接空挂在事务里）。

### 池子要调多大：按峰值倒推

默认值 15 个并发连接是起点，不是推荐值。倒推公式：目标最大并发连接数 ≈ 峰值同时到达的请求数（突发场景）或到达率 × 平均持有时间（平稳场景），再留 20% 余量。

- 平稳场景：到达率 50 请求/秒、持有 0.2 秒，并发连接约 10 个，留余量后 10 × 1.2 = 12 个，默认 15 够用。
- 突发场景：同一秒涌入 40 个请求、每请求持有 0.2 秒，峰值需要 40 个连接，默认 15 不够：可设 `pool_size=30, max_overflow=10`，上限 30 + 10 = 40，刚好兜住；再多就排队。
- 别只调 `max_overflow`：溢出连接用完即弃，高峰一过就断开重建；常态并发超过 5 个，优先调大 `pool_size`，让连接常驻复用。

### Session 生命周期：给一次业务动作定查询预算

一个 Session 对应一个业务动作（第 2 节），先给它定条数预算，超了就开 `echo=True` 数 SQL、再补 `selectinload`。ChapterContextService 聚合一章上下文的参考预算：实体 1 条 + states、relations、events、rules 各 1 条 = 5 条；再加一层嵌套集合（如每条 relation 的端点状态）预算加 1 条。

| 业务动作 | 预算 | 算法 |
|---|---|---|
| 写一章分析结果（落库） | 1 次提交 | N 行 INSERT + 1 次 commit |
| 聚合一章上下文 | 5 条 SQL | 1 条主查询 + 4 个集合各 1 条 |
| 查单个实体及全部状态 | 2 条 SQL | 1 条主查询 + 1 条 IN |

预算不是硬上限，是对账工具：实测 51 条、预算 5 条，差出来的 46 条就是 N+1。

### 边界条件

- SQLite 不走这套池算式：小说引擎用 SQLite，方言默认是 SingletonThreadPool / NullPool，没有 `pool_size + max_overflow = 15` 这道题；上面连接池的换算只在换 MySQL / PostgreSQL 后才适用，且 SQLite 是单写模型，并发写先看第 5 节。
- `selectinload` 的 IN 列表随父集线性变长：父集上万时第二条 SQL 的参数列表本身就很长，先分页取父集、再分批预加载，别一次把全表主键塞进一个 IN。
- `joinedload` 的膨胀只发生在集合侧（一对多/多对多）：多对一、一对一每父行只多 1 行，不膨胀；判断膨不膨胀先看关系基数，别一刀切。
- 本节所有毫秒数都是估算口径：fsync 约 5 毫秒、持有 200 毫秒都是假设值，换磁盘、换网络、换后端都要重测；条数、倍数与连接数三类换算不受时延假设影响，可以直接用。

## 原理剖析

本节只把原文已有章节给出的结论拆开讲清机制：不引入新概念，不补新数字，数字一律复用定量节已核验的算例；凡原文没展开的点，文中明确标注，不作定论。

### Session 与映射：为什么一个业务动作一个 Session

原文第 2 节的结论是：engine 全局只建一个，一个业务动作一个 Session，用完即关，标准写法是 `with SessionLocal() as session, session.begin():`。拆开看，这句话包含三层机制。第一，engine 内部管连接池，每处业务各建一个 engine 等于各建一套池，所以只建一个。第二，Session 是工作单元（Unit of Work），它在内存里跟踪本业务动作内对象的增、改、删，`session.begin()` 块内自动提交、异常自动回滚，出 `with` 连接归还池——提交与回滚的边界就是 Session 的边界，而不是某一条 SQL 的边界。第三，`expire_on_commit=False` 值得显式设置，是因为默认提交后对象属性过期，下次访问会触发懒加载刷新，若 Session 已关闭就抛 `DetachedInstanceError`；关掉过期后对象是提交时的快照，省了刷新，但代价是对象可能是旧快照（出处：原文第 2 节）。

原文未展开的点：Session 内部如何跟踪对象状态变化、脏检查具体在何时以何种方式比对属性，原文未展开，不作定论。原文第 7 节只给出对照结论——Session 带 identity map 与 Unit of Work，最接近 Hibernate 的 EntityManager，改了托管对象属性、commit 时自己就 flush 出 UPDATE——其底层检测方式原文未展开，不作定论。

### N+1 与加载策略：为什么条数从 51 条变成 2 条

原文第 4 节的结论是：`relationship` 默认懒加载，访问 `entity.states` 时才单独发一条查询，遍历 50 个人物各取状态就是 1 + 50 条 SQL。机制在于懒加载把“取集合”推迟到第一次访问时，每访问一个父对象就触发一次独立查询，查询次数随父对象数线性增长，而不是随数据量一次性发出。定量节把这条结论算成了确定的条数换算（出处：定量节“【算例 1】”）：聚合 50 个人物及各自的状态、每人平均 4 条状态，懒加载是 1 + 50 = 51 条 SQL；改用 `.options(selectinload(Entity.states))` 后是 1 条主查询 + 1 条 `WHERE id IN (...)` = 2 条，省下 49 条，约为原来的 1/25（51 ÷ 2 ≈ 25.5 倍）。

两种预加载的区别也在条数与行数上分得清：`selectinload` 是主查询后每集合一条 IN 查询，集合数每加 1 只加 1 条——再加一个集合（如 relations，每人平均 3 条），懒加载变成 1 + 50 + 50 = 101 条，`selectinload` 两个集合是 1 + 2 = 3 条；`joinedload` 只有 1 条 SQL，但返回行数膨胀，50 × 4 = 200 行，是父行数的 4 倍，还要 `.unique()` 去重，两层集合则是 50 × 4 × 3 = 600 行，膨胀 12 倍（出处：定量节“【算例 1】”）。所以原文第 4 节“`joinedload` 适合多对一/一对一，一对多别滥用”的结论，机制就是行膨胀只发生在集合侧（一对多/多对多），多对一、一对一每父行只多 1 行、不膨胀（出处：定量节“边界条件”）。

原文未展开的点：`selectinload` 的 IN 列表在驱动层面如何分批发出、超长 IN 列表的具体分批阈值，原文未展开，不作定论；定量节只给出边界口径——父集上万时先分页取父集、再分批预加载，别一次把全表主键塞进一个 IN。

### flush、commit 与整章一个事务：为什么提交次数是成本大头

原文第 5 节的结论是：`flush` 把变更发到数据库（拿到自增主键、触发约束检查）但未提交，`commit` 才落定；先拿 id 再建关联时手动 `flush()`，别为了拿 id 提前 commit 把业务动作拆成两个事务；分析回写整章一个事务，一章的 entities/relations/events 要么全落，要么全回滚重抽。机制在于 flush 与 commit 管的是不同阶段：flush 只发 SQL、不提交，成本是 SQL 往返而非 fsync；commit 才涉及落定与 fsync。定量节把这一点算成了确定的换算（出处：定量节“【算例 2】”）：一章解析出 1000 条 EntityState，假设每次 commit 的 fsync 约 5 毫秒（本地 SSD 估算值），逐条 commit 是 1000 次提交 × 5 毫秒 = 5000 毫秒 ≈ 5 秒，只算提交、不含 INSERT 本身；一次事务 + Core 批量 insert 是 1 次提交 × 5 毫秒 = 5 毫秒，提交成本降为 1/1000。

同一逻辑也解释了 flush 频率的换算（出处：定量节“【算例 3】”）：10000 行每 1000 行 `flush()` 一次是 10 次 flush，攒满再 flush 比每行 flush 少 9990 次往返；调用次数上，逐条 `add()` + `flush()` 是 1000 次 execute 调用，Core `insert()` 配 executemany 是一次调用带 1000 行参数，Python 侧的调用次数从 1000 降到 1，10000 行同理是 10000 次调用降到 1 次。需要主键回填时，2.0 的 `insertmanyvalues` 默认每页 1000 行：1000 行 = 1 页、1 次往返取回主键，10000 行 = 10 页、10 次往返（`ceil(10000 ÷ 1000) = 10`）（出处：定量节“【算例 2】”）。原文第 8 节“循环里逐个 commit 性能差且破坏原子性”的结论，机制正在这里：逐个 commit 既把提交次数乘到 N 倍，又把一个业务动作拆成 N 个事务，落一半的世界状态比不落更糟。

原文未展开的点：fsync 约 5 毫秒只是本地 SSD 估算值，换磁盘、换网络、换后端都要重测，定量节已明确毫秒数是估算口径，条数、倍数与连接数三类换算不受时延假设影响；WAL 模式下读写互不阻塞的具体实现机制，原文第 5 节只给出“用 `event` 监听 connect 事件执行 `PRAGMA journal_mode=WAL` 开 WAL”的做法，未展开，不作定论。

### 连接池与 Session 生命周期：为什么 15 是上限、5 条是预算

原文定量节给出的连接池结论是确定的算术，不是经验值（出处：定量节“连接池：最大连接数是一道算术题”）：QueuePool 默认值 `pool_size = 5`，`max_overflow = 10`，`pool_timeout = 30` 秒，最大并发连接数 = 5 + 10 = 15，第 16 个并发请求起排队等连接，超过 30 秒拿不到就报超时；溢出连接用完即断开丢弃，只有 `pool_size` 的 5 个会常驻池中。定量节“【算例 4】”把排队过程算清了：20 个并发请求同时到达、每请求持有连接 200 毫秒，前 15 个立即拿到连接，剩下 5 个排队，按每批 15 个、每批 200 毫秒算，2 批约 0.4 秒消化完，无人超时；同样 20 个并发、每请求持有连接 5 秒，第 16–20 个要等约 5 秒才能拿到连接（仍在 30 秒超时内）；持有时间涨到 30 秒以上（如 40 秒）才会有人排队超 30 秒报超时，排队等待约等于一个持有时长。

池够不够用按 Little 定律估（出处：定量节“【算例 4】”）：并发连接数 ≈ 到达率 × 平均持有时间，50 请求/秒 × 0.2 秒 = 10 个连接，15 的上限够用；50 请求/秒 × 0.5 秒 = 25 个，超了 15，要么调大池，要么缩短持有时间（Session 用完即关，别让连接空挂在事务里）。倒推时再留 20% 余量：平稳场景 10 × 1.2 = 12 个，默认 15 够用；突发场景同一秒涌入 40 个请求、每请求持有 0.2 秒，峰值需要 40 个连接，可设 `pool_size=30, max_overflow=10`，上限 30 + 10 = 40，刚好兜住（出处：定量节“池子要调多大：按峰值倒推”）。

Session 生命周期与查询预算是同一套对账逻辑（出处：定量节“Session 生命周期：给一次业务动作定查询预算”）：一个 Session 对应一个业务动作，先定条数预算，超了就开 `echo=True` 数 SQL、再补 `selectinload`；ChapterContextService 聚合一章上下文的参考预算是实体 1 条 + states、relations、events、rules 各 1 条 = 5 条，写一章分析结果是 1 次提交，查单个实体及全部状态是 2 条 SQL；预算不是硬上限，是对账工具，实测 51 条、预算 5 条，差出来的 46 条就是 N+1。

原文未展开的点：QueuePool 内部排队的唤醒顺序与溢出连接重建的细节，原文未展开，不作定论；且上述池算式只在换 MySQL / PostgreSQL 后才适用，小说引擎用的 SQLite 默认是 SingletonThreadPool / NullPool，没有 `pool_size + max_overflow = 15` 这道题，SQLite 是单写模型，并发写先看原文第 5 节（出处：定量节“边界条件”）。

## 实战案例

### 案例一：聚合一章上下文，SQL 条数从 51 条压到预算 5 条

现象：ChapterContextService 聚合一章上下文，遍历 50 个人物各取状态，实测 SQL 条数远超预算。按定量节的预算表，聚合一章上下文的参考预算是 5 条（实体 1 条 + states、relations、events、rules 各 1 条），而懒加载下仅 states 一个集合就是 1 + 50 = 51 条 SQL，实测 51 条、预算 5 条，差出来的 46 条就是 N+1（出处：定量节“【算例 1】”与“Session 生命周期：给一次业务动作定查询预算”）。

排查：按原文第 4 节的排查手段，开 `echo=True` 数 SQL 条数，N+1 是一排同构 SQL；排查期也可用 `raiseload` 策略让懒加载访问直接报错，定位是哪些访问在逐个触发查询。数完条数再对照预算表：主查询 1 条是对的，多出来的 50 条是 50 个人物各触发的 1 条 states 查询，与懒加载“访问时单查”的机制一致。

结论：批量场景按需改预加载，如 `.options(selectinload(Entity.states))`，用一条额外 IN 查询把集合批量带出。按定量节“【算例 1】”的换算，50 个人物、每人平均 4 条状态时，`selectinload(Entity.states)` 是 1 条主查询 + 1 条 `WHERE id IN (...)` = 2 条，省下 49 条，约为原来的 1/25（51 ÷ 2 ≈ 25.5 倍）；集合数每加 1，`selectinload` 只加 1 条，聚合入口的条数预算直接按 `1 + 集合数` 算，4 个集合就是 5 条，与预算表一致。不要改用 `joinedload` 治这个场景：它是 1 条 SQL 但返回 50 × 4 = 200 行，行数是父行数的 4 倍，还要 `.unique()` 去重（出处：定量节“【算例 1】”）。

### 案例二：一章 1000 条状态落库，提交次数从 1000 次压到 1 次

现象：一章解析出 1000 条 EntityState 落库，循环里逐个 commit，又慢又破坏原子性。按原文第 8 节，这属于“循环里逐个 commit”误区；按原文第 5 节，分析回写应整章一个事务，一章的 entities/relations/events 要么全落，要么全回滚重抽。

排查：先数的不是 INSERT 条数，而是提交次数。逐条 `add()` + 逐条 `commit()` 时，N 行就是 N 次 commit，每次 commit 至少一次往返 + 一次 fsync，成本不在 INSERT 本身而在提交次数。按定量节“【算例 2】”的换算，假设每次 commit 的 fsync 约 5 毫秒（本地 SSD 估算值），1000 次提交 × 5 毫秒 = 5000 毫秒 ≈ 5 秒，只算提交、不含 INSERT 本身。

结论：一次事务收口（原文第 5 节口径），把提交次数压到 1 次，再走 Core `insert()` 的 executemany，把 N 次 INSERT 调用合并成少量往返。按定量节“【算例 2】”的换算，一次事务 + Core 批量 insert 是 1 次提交 × 5 毫秒 = 5 毫秒，提交成本降为 1/1000；调用次数上，逐条 `add()` + `flush()` 是 1000 次 execute 调用，Core `insert()` 配 executemany 是一次调用带 1000 行参数。代码只复用原文已出现的写法：

```python
with SessionLocal() as session, session.begin():
    session.execute(insert(Entity), [{...}, {...}])
```

需要主键回填时，按定量节“【算例 2】”，2.0 的 `insertmanyvalues` 默认每页 1000 行，1000 行 = 1 页、1 次往返取回主键。原文未展开的点：`bulk_save_objects` 与 Core insert 跳过 ORM 状态跟踪后、对象不进 identity map 的后续影响，定量节只给出“对象不进 identity map”这一结论、未展开其排查含义，不作定论；批量路径的取舍以定量节原文为准。

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

4. 一章要落 1000 条状态、聚合 50 个人物及各自的状态：批量落库与查询各要几条 SQL、连接池 pool_size=5、max_overflow=10 时 20 个并发请求会怎样？

### 参考答案

1. 因为 relationship 默认懒加载：访问 entity.states 时才单独发一条查询，遍历 50 个人物就是 1 + 50 条，这就是经典 N+1。批量场景改预加载，如 .options(selectinload(Entity.states))，用一条额外 IN 查询把集合批量带出；joinedload 是 JOIN 一次带出，适合多对一/一对一，一对多会行膨胀，别滥用。排查手段是开 echo=True 数 SQL 条数，N+1 是一排同构 SQL；raiseload 可让懒加载访问直接报错，排查期好用。
2. flush 只是把变更发到数据库（拿到自增主键、触发约束检查）但未提交，commit 才落定。先拿 id 再建关联时手动 flush()，别为了拿 id 提前 commit 把业务动作拆成两个事务；批量落库也要一次事务收口，不能循环里逐个 commit。
3. 因为默认 expire_on_commit=True，提交后对象属性过期，下次访问触发懒加载刷新，而 Session 已关闭就抛 DetachedInstanceError。避法是显式设置 expire_on_commit=False（代价是对象可能是旧快照），或按第 8 节的做法：服务层出口转 dict / dataclass 再传，不要把 ORM 对象当 DTO 跨层乱传。

4. 懒加载 51 条，selectinload 2 条；joinedload 1 条但 200 行、膨胀 4 倍。逐条 commit 1000 次，一次事务+Core insert 仅 1 次，5 秒降至 5ms。池上限 15，20 并发 5 个排队，持有 200ms 等 0.2s，超 30s 报超时。

## 9. 关联知识

### 前置知识

- [Python](开发语言/Python/Python.md)：语言基础与总索引，本篇的映射与 Session 写法都建立在 Python 语法与面向对象基础之上（引用原文第 2 节）
- [pydantic 数据校验](开发语言/Python/pydantic.md)：落库前校验形状，SQLAlchemy 管落库，引擎输入是 LLM 输出、本质不可信，校验管形状、参数化管注入（引用原文第 6 节）
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：服务层出口转 dataclass 传递，不要把 ORM 对象当 DTO 跨层乱传（引用原文第 8 节）
- [MySQL-数据库](数据库/MySQL/MySQL-数据库.md)：SQL 与事务基础，换引擎不换原理，连接池算式只在换 MySQL / PostgreSQL 后才适用（引用原文定量节“边界条件”）

- [Python](开发语言/Python/Python.md)：语言基础与总索引
- [pydantic 数据校验](开发语言/Python/pydantic.md)：落库前校验形状，SQLAlchemy 管落库
- [dataclass 与 typing](开发语言/Python/dataclass与typing.md)：服务层出口转 dataclass 传递
- [MySQL-数据库](数据库/MySQL/MySQL-数据库.md)：SQL 与事务基础，换引擎不换原理

## 10. 一句话总结

SQLAlchemy = 一半 MyBatis 的可控 + 一半 Hibernate 的托管：Core 管得住 SQL，ORM 省得掉样板，Session 是两者共同的事务边界。
