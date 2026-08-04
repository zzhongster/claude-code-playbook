# 反模式：报错信息本身建议的那个修法，把 fail-loud 改成了 fail-silent

## 一句话结论

工具报错时往往同时暗示了「怎么让它不报」，而那个修法**经常不是解决问题，只是关掉报警**——命令从此退出码 0、产物照常生成、下游校验照常通过，唯独你要的东西没了。**报错文本里出现的选项名，是最危险的一类修改建议**：它看起来像官方指路，实际是把守卫拆掉的开关。

## 场景

- 某个操作因为权限 / 校验 / 一致性检查而**明确报错退出**（这是好的）
- 报错文本里提到一个开关、参数或配置名（`--enable-*`、`--force`、`--no-verify`、`ignore_errors`、`continue-on-error`…）
- 加上它之后命令立刻变绿
- **没有人检查产物内容变了没有**——因为「命令成功了」

AI 尤其容易踩：报错文本是最强的上下文信号，模型会顺着它给出「最小改动让它跑通」的方案，而「跑通」正是被优化的目标。

## 详细说明

### 踩坑经历

TOUCH 2.0 项目（2026-08-04），给生产库备份做最小权限收窄：把 `pg_dump` 用的库主账号换成只读角色。业务表按规格全部启用了 `FORCE ROW LEVEL SECURITY`（多租户隔离）。

只读角色跑 `pg_dump`，Postgres 明确拒绝：

```
pg_dump: error: query failed: ERROR:  query would be affected by row-level security policy for table "lead_list"
pg_dump: detail: Query was: COPY public.lead_list (id, tenant_id, name) TO stdout;
```

退出码 1。**这是设计得很好的失败**：PG 知道它给不出完整数据，于是拒绝给出任何数据。

报错说「会被 RLS 影响」。最顺手的修法就是把 RLS 打开——`pg_dump` 恰好有这个参数：

```bash
pg_dump --enable-row-security ...
```

加上之后**退出码 0**。实测产物：

| 项 | 结果 |
|---|---|
| 退出码 | 0 |
| `CREATE TABLE` / `CREATE POLICY` / `ENABLE ROW LEVEL SECURITY` | 全在 |
| **数据行** | **0 行** |

一份结构完整、数据全空、看起来完全正常的备份。

**而下游的验收也过了。** 那条备份链路有「恢复演练」：起一个临时库把 dump 导进去，确认能恢复。它查的是

```sql
select count(*) from information_schema.tables where table_schema not in ('pg_catalog','information_schema')
```

**表的数量**。全空的 dump 导进去零报错，表数正常，日志打「恢复后的用户表数量：N」，**全绿**。

于是三道防线依次失效：PG 的 fail-loud 被参数关掉 → 产物看起来正常 → 验收查错了观测量。而这一切要到**真正需要恢复数据的那天**才显形，那天已经没有补救余地。

### 为什么这条比「一般的错误修复」更隐蔽

普通的错误修复，改错了会继续报错。这一类的特征是**改完就没有任何反馈了**：

- 报错消失 = 唯一的信号消失
- 退出码 0 = 自动化认为成功
- 产物存在且格式合法 = 肉眼抽查也看不出

**它把「有信号的失败」变成了「无信号的失败」**，而后者在所有失败形态里是最贵的。

### 正确的修法长什么样

回到那个报错真正在说什么：「我读不全这张表」。有两条路——

- **让它读得全**（本例：给角色 `BYPASSRLS`，或用 `pg_read_all_data` + `BYPASSRLS`）；
- **接受读不全，但要显式声明**（真的只想导某租户的数据时才用 `--enable-row-security`，且必须断言行数符合预期）。

`--enable-row-security` 本身不是坏参数，它有正当用途。**坏的是把它当作「消除报错」的手段而不是「表达意图」的手段。**

## 数据支撑

同一张表、两个租户各一行，PG 18：

| 角色 | 参数 | 退出码 | dump 中数据行 |
|---|---|---|---|
| 无 `BYPASSRLS` | 默认 | **1** | —（拒绝输出） |
| 无 `BYPASSRLS` | `--enable-row-security` | **0** | **0** |
| 有 `BYPASSRLS` | 默认 | 0 | 2（完整） |

## 怎么防

1. **看到报错里的选项名，先问「它是让我读得全，还是让我不再被告知读不全」。** 这个问题能一句话分开两类修法。
2. **改完之后比对产物，不是比对退出码。** 本例只要数一次行数就露馅。凡是「加了个参数就绿了」，产物必须重新量。
3. **给守卫写反例用例。** 防线失效时整条链路仍然全绿，所以防线本身必须被单独测——造一份全空 dump，断言它**过不了**验收。这条与 shell 里单独测 `set -o pipefail`、psql 单独测 `ON_ERROR_STOP=1` 是同一件事。
4. **把理由写在开关旁边，不是写在文档里。** 下一个改这行的人不会读你的设计文档，但一定会读他正在改的那个文件。本例把「为什么必须 BYPASSRLS、为什么禁止 `--enable-row-security`」直接写进角色定义的注释。

## 适用范围

- 适用于：任何「校验失败 → 有开关可以跳过校验」的组合。常见面孔：`pg_dump --enable-row-security`、`git commit --no-verify`、`curl -k`、`npm install --force`、`tar --ignore-failed-read`、CI 的 `continue-on-error: true`、`psql` 不加 `ON_ERROR_STOP`、`set -o pipefail` 缺失。
- 不适用于：报错本身就是误报、且已确认产物完整的情况——那时跳过校验是对的，但**「已确认产物完整」这一步不能省**。

## 相关

- [green-metric-measures-wrong-mechanism.md](green-metric-measures-wrong-mechanism.md) —— 近亲。那条讲**指标在故障态恰好返回正常值**；这条讲**修法主动把故障态变成了正常值**。本例两者同时发生：参数消掉了报错，而验收指标（数表不数行）又接不住。
- [tdd-fake-red.md](tdd-fake-red.md) —— 测试绕开了 bug 的触发条件。
