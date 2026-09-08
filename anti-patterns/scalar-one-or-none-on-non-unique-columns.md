# 反模式：在没有唯一约束的列上用 `scalar_one_or_none()`，脏数据一到就整批回滚

## 一句话结论

`scalar_one_or_none()` / `one()` 是一句断言："这个条件最多命中一行"。断言只有在数据库用唯一约束替你担保时才成立；靠"业务上应该唯一"来担保，迟早有一次合库、一次并发、一次手工导入把它打破，然后这条查询会把整个事务连同几十分钟的上游工作一起拖下水。

## 场景

- ORM 里按业务键（公司名 + 国家、邮箱、外部 ID）查"有没有这条"，习惯性写 `scalar_one_or_none()`
- 该业务键从未建过 unique 索引，只是"大家都知道它应该唯一"
- 查询嵌在一个大事务的循环里，前面是昂贵的工作（LLM 调用、抓取、批量计算），后面才是落库

## 详细说明

### 踩坑经历（trade-ai #431，2026-09-08）

买家推荐任务：AI 跑 15 轮搜索（30-50 分钟）→ 拿到几十家公司 → 循环入库。入库循环按 `(name, country)` 查 `companies` 表：

```python
existing = await db.execute(
    select(Company).where(Company.name == name, Company.country == country)
)
company = existing.scalar_one_or_none()   # ← 断言：最多一行
```

`companies` 从 3 月建表起就没有 `(name, country)` 唯一约束。7 月生产机迁移期间两台机器双跑了三周，合库时按新 id 追加、没按业务键去重，留下 45 对重复行。之后只要 AI 结果里出现这 45 家中的任意一家（AutoZone México、Kiabi、Lenta……），就抛 `MultipleResultsFound`，外层 `except` 把任务标 failed，事务回滚，**0 条落库**。

08-05 到 09-08 挂了 9 个任务、3 个用户；其中一人连挂 7 次——因为他反复重跑同一个法国方向，而重复行有 20 多家是法国饰品品牌。每次重跑再烧一遍几十分钟的 LLM，再挂一次。

用户看到的错误是 `Multiple rows were found when one or none was required`，跟"AI 搜索"八竿子打不着，一线根本猜不到是数据问题。

### 为什么会反复出现而不是一次性

- 断言写在**读**路径，但破坏它的是**写**路径（合库脚本、并发插入、手工导入）——两边的作者互不知道对方在依赖什么
- 重复行一旦存在就是永久地雷：谁的搜索方向命中它谁倒霉，跟代码改动无关，所以看起来"随机"
- 没有唯一约束，第二类来源（并发竞态）还会持续制造新地雷：同一个业务库里 3 月就有一对是两个任务相隔 3 分钟各插一行

### 替代方案

三层一起做，少一层都会再犯：

1. **查询不再断言**：`order_by(id).limit(1)` + `scalars().first()`，永远不抛 `MultipleResultsFound`。脏数据再脏，最多是取到"较老的那一行"，不会让上游几十分钟的工作归零。
2. **把担保交给数据库**：按业务键去重（保留最小 id、引用改指、空字段互补）后建唯一索引。清数据不加约束 = 过几个月再来一次。
3. **写路径接住约束**：加了唯一索引后，并发插同一条会从"多一行重复"变成 `IntegrityError`。get-or-create 必须在 savepoint 里 flush，撞索引就取已存在行——见 [savepoint 内 get-or-create](../patterns/savepoint-get-or-create.md)。

### 自检

```sql
-- 任何被 scalar_one_or_none() 按业务键查的表，跑一次：
SELECT col_a, col_b, count(*) FROM t GROUP BY 1, 2 HAVING count(*) > 1;
```

```bash
# 代码侧：把每个 scalar_one_or_none / .one() 的 where 条件对照一遍 unique 约束
grep -rn "scalar_one_or_none\|\.one()" app/ | grep -v "\.id ==\|primary"
```

顺带一个更隐蔽的变体：`ilike('%name%')` 模糊匹配后接 `scalar_one_or_none()`——列本身唯一，但子串匹配天然多命中。

## 适用范围

- 适用于：SQLAlchemy / Django ORM `.get()` / 任何"取唯一一行否则抛"的 API；尤其是长事务、昂贵上游的入库循环
- 不适用于：按主键或已有 unique 索引的列查——那里断言是对的，该抛就抛

## 相关

- [savepoint 内 get-or-create](../patterns/savepoint-get-or-create.md)
- trade-ai Issue #431 / PR #432
