# 模式：get-or-create 在 savepoint 里 flush，撞唯一索引就取已存在行

## 一句话结论

"先查再插"在并发下永远有窗口；正确的 get-or-create 是**先查 → 没有就在 savepoint 里插并 flush → 撞唯一索引则回滚 savepoint、再查一次取已存在行**。外层事务不受影响，调用方拿到的一定是一行可用的记录。

## 场景

- 多个后台任务 / 请求并发地往同一张"字典表"写同一条业务键（公司、标签、实体、外部 ID）
- 该表有（或刚加上）业务键唯一索引
- 写入发生在一个大事务里，失败不能把整个事务拖死

## 详细说明

### 为什么"先查再插"不够

```python
company = await find_company(db, name, country)   # 两个任务同时查到 None
if not company:
    company = Company(...)
    db.add(company)
    await db.flush()   # 第二个任务在这里撞唯一索引 → IntegrityError → PG 事务进入 aborted 状态
```

PostgreSQL 里一旦某条语句报错，整个事务就进入 aborted 状态，后续任何语句都是 `current transaction is aborted`。没有 savepoint，唯一的出路是整个事务回滚——上游几十分钟的结果全丢。

### 写法（SQLAlchemy 2.x async）

```python
async def find_company(db, name, country) -> Company | None:
    """按业务键查；历史脏数据可能有重复行，固定取最小 id，绝不抛 MultipleResultsFound。"""
    result = await db.execute(
        select(Company)
        .where(Company.name == name, Company.country == country)
        .order_by(Company.id)
        .limit(1)
    )
    return result.scalars().first()


async def insert_company_or_get_existing(db, company: Company) -> Company:
    """插入并 flush 拿 id；撞 (name, country) 唯一索引时改为返回已存在的那行。"""
    try:
        async with db.begin_nested():      # SAVEPOINT
            db.add(company)
            await db.flush()
        return company
    except IntegrityError:                 # savepoint 已自动回滚，外层事务仍然活着
        existing = await find_company(db, company.name, company.country)
        if existing is None:
            raise                          # 不是业务键冲突而是别的约束，不能吞
        return existing
```

调用方：

```python
company = await find_company(db, name, country)
if company is None:
    company = await insert_company_or_get_existing(db, Company(name=name, country=country, ...))
# 之后一律用返回值——并发场景下它可能不是你 new 出来的那个对象
```

### 几个细节

- `begin_nested()` 作为上下文管理器：块内异常 → 自动 `ROLLBACK TO SAVEPOINT`，块内 `add` 的对象被 expunge；块正常退出 → `RELEASE SAVEPOINT`
- `except` 里必须**再查一次**而不是返回传入对象——传入对象已经被 expunge，没有 id
- `existing is None` 时 re-raise：唯一索引冲突之外的 IntegrityError（外键、非空）不能被这个兜底吞掉
- 查询用 `order_by(id).limit(1)` 而不是 `scalar_one_or_none()`：即使唯一索引还没建（或将来被人删了），这条路径也不会炸——见反模式 [在没有唯一约束的列上用 scalar_one_or_none](../anti-patterns/scalar-one-or-none-on-non-unique-columns.md)

### 验证

在 trade-ai #431 里两套环境都跑过：

| 环境 | 结果 |
|---|---|
| sqlite + aiosqlite（单测） | 撞索引返回已存在行；同 session 继续插别的记录并 commit 正常 |
| PostgreSQL + asyncpg（本地 docker） | 同上；savepoint 回滚后事务可继续写 |

## 数据支撑

trade-ai 生产 `companies` 表在没有唯一索引的 6 个月里，靠"先查再插"产生了 1 对并发重复（两任务相隔 3 分钟各插一行）；合库又带来 44 对。加索引 + 本模式后，并发路径的结果从"多一行地雷"变成"拿到同一行"。

## 适用范围

- 适用于：字典表 / 实体表的 get-or-create；有唯一索引；写入在长事务内，不允许整体回滚
- 不适用于：可以接受整事务重试的短请求（直接 `INSERT ... ON CONFLICT DO NOTHING RETURNING` 更简单）；MySQL 老版本对 savepoint 支持有差异需单独验证

## 相关

- [在没有唯一约束的列上用 scalar_one_or_none](../anti-patterns/scalar-one-or-none-on-non-unique-columns.md)
- trade-ai Issue #431 / PR #432（`server/app/services/company_lookup.py`）
