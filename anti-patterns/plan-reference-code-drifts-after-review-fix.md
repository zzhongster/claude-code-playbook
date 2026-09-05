# 反模式：评审修复只改实现，计划文档里那份参考代码原样留着——漂移方向永远偏向"漏洞留存"

## 一句话结论

在"计划里先写好完整可抄的参考代码 → 实现者转写 → review 修实现"这类流程里（superpowers SDD、spec-driven、任何带 implementation plan 的 agent 工作流），**review 修的是 `src/`，计划文件不在 diff 里、没人认领**。于是仓库中会留下一份以"Step 3: 实现"权威形式给出的、已被实测证明有缺陷的参考实现——而漂移方向是单向的：**修复留在代码里，缺陷留在文档里**，专等下一个人照抄。

## 场景

- 计划/spec 里为了让实现者"抄了就对"，写了完整的实现代码与测试代码
- 实现者忠实转写并提交，review 在**实现的 diff** 上发现缺陷
- 修复补丁改的是 `src/` 与 `tests/`，计划文件在上一个提交里、不在本次 diff 的范围内
- 于是同一个仓库里出现两份同名实现：代码是修好的，文档是原来的

## 踩坑经历

2026-09-05，`trade-agent-data` 的 D7 薄片（内部运营后台）。计划文件 `docs/superpowers/plans/2026-09-03-d7-admin-console.md` 的 Task 1 给了 `AdminStore` 的完整实现，其中：

```python
    def promote(self, sid: str, userid: str) -> None:
        with closing(self._conn()) as conn, conn as c:
            c.execute("UPDATE admin_sessions SET userid=?, expires_at=?, last_seen=?"
                      " WHERE sid_hash=?", (userid, _shift(SESSION_TTL), _now(), hash_key(sid)))
```

整分支终审（opus）在**实现的 diff** 上实测出四个问题，其中两个是安全级的：

| 输入 | 实测结果 | 应该 |
|---|---|---|
| 对**已过期**的待认证行 promote | 复活成 8 小时有效会话 | 拒绝 |
| 对**已是正式会话**的 sid promote 成另一个 userid | 静默换人（角色一并升级） | 拒绝 |
| 跨认证边界不换 sid | **session fixation**：攻击者自己走一遍 `/login` 拿到未绑定的 sid，塞进受害者浏览器，受害者认证后该 sid 即成为攻击者的已认证会话 | 换发新 sid |

修复很干净：`promote()` 改成带守卫的会话轮换（`WHERE ... AND userid IS NULL AND expires_at > ?`，换发新 `sid_hash`，返回 `str | None`），补了五条守卫测试，spec 里那句"回调校验通过后同一行写入 userid"也同步改成了"换发新 sid"。

**但计划文件里那段 `promote()` 原封不动。** 一轮 scoped re-review 抓到了它，判定 **Minor（文档一致性问题，不影响已合入的生产代码）**。

判 Minor 的理由站得住——它确实不影响任何运行中的代码。但漏掉了三件事：

1. 那段代码不是"描述"，是**指令**：它出现在标题为 `Step 3: 写最小实现` 的代码块里，形式上就是"照这个写"
2. 同一份计划**还有六个任务没做**（卡在一个外部前置上），后续任务的实现者必须读这份计划（要看 Task 1 的 Interfaces 才知道 `AdminStore` 长什么样）
3. 分支回滚重做、或新人接手照抄，抄到的就是那个已被实测证明可绕过认证的版本

最终由控制方（计划作者）裁决为"比 Minor 重"，单独开工同步了八处：`promote` 的守卫与轮换、`_shift(delta)` → `_shift(base, delta)`（与姊妹模块签名对齐）、`_audit_in` → 公开的 `audit_in`、以及三条仍在用旧调用方式的测试。

## 为什么漂移方向是单向的

这不是随机的"文档过期"，而是流程结构决定的：

- **review 的输入是 diff**。计划文件是上一个提交的产物，天然在 diff 之外——reviewer 想看也得跨出范围
- **修复的执行者是实现者**，它的任务定义是"修代码里的 finding"，改文档不在它的范围里，改了反而算范围蔓延
- **只有缺陷会触发修复**。计划里写对的部分没人动，写错的部分被实现侧修掉了——文档侧留下来的**恰好是被判定为错的那一份**

所以每一轮 review 都在扩大"代码正确、文档危险"的落差，且落差里装的全是安全相关的内容（因为只有严重问题才会进 fix loop）。

## 怎么防

**流程上**：把"计划/spec 里的对应代码块与签名"写进 fix wave 的显式条目。本次终审其实做对了一半——B1/B2 两条专门修 spec 与计划里被推翻的论断，但只覆盖了"Task 4 的调用方代码"，漏了"Task 1 已完成任务的演练稿"，因为潜意识里觉得"做完的任务不用管了"。**已完成任务的计划正文同样会被读**。

**机制上**：签名漂移可以机器检测。计划里的代码块与 `src/` 的实际实现，方法名与签名应该能对上：

```bash
# 计划里出现的符号 vs 实际实现的签名，人工比对一次胜过读三遍
grep -n "def promote\|def _shift\|_audit_in" docs/superpowers/plans/<plan>.md
grep -n "def promote\|def _shift\|def audit_in" src/<module>/store.py
```

**结构上**（更根本，但有代价）：计划里只写**接口契约 + 约束 + 测试**，不写完整实现体。实现者自己写实现，就不存在"文档里的实现"这个东西。代价是计划的可执行性下降、实现者自由发挥的空间变大——在 agent 驱动的流程里这个代价通常不划算，所以更实际的还是"把文档同步写进修复范围"。

**判断严重度时**：文档不一致默认是 Minor，但当文档内容满足"会被照抄 + 涉及安全/正确性 + 存在还没做完的下游任务"三条时，应该升级。它的危害不在今天的代码里，在下一个人的键盘上。

## 适用范围

- **适用于**：任何在文档里写完整参考实现的流程——implementation plan、spec 里的示例代码、README 里的 quickstart 片段、runbook 里的命令行。尤其是多薄片/多阶段项目，后续阶段的执行者会重读同一份文档
- **不适用于**：文档只描述意图与契约、不给可抄代码的项目——那里没有这个漂移面
- **更通用的教训**：**任何"可执行文档"在代码被修复后都会漂，而且漂的方向偏危险**——因为只有错的那部分才会在代码侧被改掉。修复的范围定义应该问一句"这个缺陷的副本还存在于哪里"，而不只是"缺陷在代码的哪一行"

## 相关

- 项目源：`trade-agent-data`，D7 薄片（Linear YC3-32），2026-09-05（commit `cec1161` 同步八处）
- 同类：[guard-passes-without-the-fix.md](guard-passes-without-the-fix.md)（守卫绿了但修复没落地）、[same-session-self-review.md](same-session-self-review.md)（自审的结构性盲区）、[evidence-without-discriminating-power.md](evidence-without-discriminating-power.md)（证据没有区分力）
