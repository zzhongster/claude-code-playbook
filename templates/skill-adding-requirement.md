# 模板：adding-requirement —— 冻结期「记录一条需求」的技能

> 安装：整段 SKILL.md 落到 `~/.claude/skills/adding-requirement/SKILL.md`（个人技能，不入项目仓）。
> 来源：TOUCH 2.0（touch2-engine）2026-09-10，按 TDD 立（先跑无技能基线、再写、再对照），数据见 [experiments/2026-09-10-skill-tdd-baseline-vs-with-skill.md](../experiments/2026-09-10-skill-tdd-baseline-vs-with-skill.md)。

## 它解决什么

功能冻结后，负责人从客户现场带回来的「观察 + 想法」怎么记。两个失败形态都在基线里出现过：**把转述当事实直接记**（没核前提），以及**为了记一条需求去读实现代码**（63 次工具调用）。技能把「记需求」钉成三步：核前提 → 查复发 → 落卡；卡面是写给几个月后不认识你的人读的。

## 移植时要换的东西

| 技能里的写法 | 换成你项目的 |
|---|---|
| `ycy_touch` / `TOUCH 2.0` / Backlog / Low / Feature | 你的 Linear（或 Jira）team、project、状态、标签 |
| `web-access` 技能 | 你手上能亲眼看网页 DOM 的工具 |
| 易查云 MCP `search_trade` / `find_buyers` | 你领域里的一手数据源（不是搜索引擎） |
| `docs/dev-spec/`、`docs/compliance/`、`07 §5` | 你项目的规格分册与「已知缺口」登记处 |
| 铁律 5 / `docs/sessions/` / `Part of TOU-<n>` | 你项目的会话归档约定；没有就删掉第 4 步 |

「红旗」表里的两行实据（0/10、63 次）是本项目的，换成你自己跑基线得到的数字——**没跑基线就别写数字**。

---

## SKILL.md 原文

````markdown
---
name: adding-requirement
description: Use when the user hands over a product idea, a customer or field observation, a competitor behavior, or says 记录这个需求 / 加进需求池 / 以后版本可能加 / TOUCH 未来可能做——especially after feature freeze, and especially when the idea rides on an unverified claim (a LinkedIn profile, "大部分都是…", "客户说…"). Also use before creating any Linear card for a feature that has no owner and no schedule.
---

# 需求入池（adding-requirement）

## 核心

**没核过的观察是传闻，不是需求。** 进池的东西 = 核过的观察 + 候选需求 + 裁决点。卡面是写给几个月后不认识你的人读的：他要能**不重做调研**就判断值不值得开。

## 用 / 不用

- 用：用户给一条观察、想法、竞品行为，要「记下来」「以后加」「未来版本」。
- 不用：已有卡要排期（`/start-task`）；bug（直接开 Bug 卡）；改规格（回源仓，`docs/SPEC-SYNC.md`）。

## 流程（顺序不跳）

### 0 · 定边界

时间盒：工具时间 ≤ 30 分钟。读到「放哪个模块 / 有没有已有 FR / 碰不碰红线」即停。**不读实现代码、不读 adapter、不设计方案**——那是开做那张卡的事。

### 1 · 核前提（必做，不许跳）

把观察拆成可核的断言，每条找**一手来源**核一次：

| 断言类型 | 去哪核 |
|---|---|
| 网页 / 档案 / 界面上「有什么」 | `web-access` 技能，亲眼看 DOM，别信转述 |
| 谁在买、买什么、从谁买 | 易查云 MCP（`search_trade` / `find_buyers`），不用网页二手数字 |
| 产品今天能不能做到 | `docs/dev-spec/` 分册（04 契约 / 06 adapter / 03 模型），只查分册不读实现 |
| 合规能不能碰 | `docs/compliance/` 与 `00-context-pack.md` §12，只引用结论 |

写下四样：核了什么、样本量 n、三态结论（成立 / 不成立 / 部分成立）、**结论改没改需求的定位**（如「买家」→「目标账户」）。n=1 只能定性，卡面必须写明「不能外推命中率」。

### 2 · 查复发（必做）

- Linear `list_issues`（team `ycy_touch`）：中文关键词、英文关键词、同义词各搜一次；
- `git grep -i "<关键词>" docs .claude`；
- 命中 → 往那张卡**追加评论**（只写新增的核验结论与订正），**不建第二张**。

### 3 · 落卡

固定值（TOUCH 2.0）：

| 字段 | 值 |
|---|---|
| team / project | `ycy_touch` / `TOUCH 2.0` |
| state / priority / assignee | Backlog / Low(4) / 不指派 |
| label | Feature |
| 标题 | `需求池 · <一句话>——<核出来的定性>（上线后再议）` |
| links | 样本 URL 作附件；**不进正文、不进仓库** |

正文按下面模板；缺的节写「未查」，不删节。建完把模板末尾的归档路径补成真卡号。

### 4 · 落归档（铁律 5）

`docs/sessions/<MMDD>-tou<n>.md` 五项固定标题；docs-only 分支 `t2-tou<n>-archive`；PR 正文写 `Part of TOU-<n>`（别让卡被翻 Done）+「偏离卡面之处：无」一节。**仓库文件不出现个人档案 URL、真实客户名**——写「见卡面附件」。

## 卡面模板

```markdown
> **需求池卡**：不排期、不指派、不进当前范围。功能冻结后新需求只落卡面，等上线后由人裁决要不要开卡。

## 一句话
<要什么> + <核出来的信号定性>

## 来源
- 提出：<谁>，<日期>，<场合>
- 原话：「…」
- 样本：见附件链接（<一句去标识的描述>）

## 验证结论（<日期>现取，n=<N>）
核了什么 / 一手来源是什么 / 三态结论 / 定位有没有变 / 样本量限制

## 需求描述（候选，未裁）
输入 / 输出 / 评分权重（相对海关成交记录）/ 呈现（source_refs、inferred 标记）

## 非目标 / 红线
禁区命中项 / 合规未评估项 / 「不动当前范围」

## 对应规格位置
07 的 FR 模块码；无 FR → §5 缺口类，开做先回源仓登记再同步

## 裁决点（上线后）
1. …（带成本估计）
2. …

## 会话归档
`docs/sessions/<MMDD>-tou<n>.md`
```

## 红旗（出现即停）

| 想法 | 现实 |
|---|---|
| 「用户说的，不用核」 | 用户是转述者不是来源。0910 样本：转述「大部分是买家」，海关数据 0/10 |
| 「先记下来，回头再核」 | 回头没有人核；没核的卡半年后被当事实引用 |
| 「顺手看看 adapter 能不能做」 | 那是开做时的事；读实现 = 时间盒炸掉（0910 基线：63 次工具调用只为记一条） |
| 「把方案也写上更完整」 | 方案会被当成决策；冻结期只留裁决点 |
| 「直接登记进 07 分册 §5」 | 镜像只读；回源仓 + `Part of` |
| 「URL 写进归档方便找」 | 禁区；附件在 Linear |
````
