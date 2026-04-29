# Pencil MCP 多屏设计协作工作流

> 用 Pencil MCP 跟 AI 一起做完整产品 UI 设计的工作流。重点解决多屏管理 / 设计追踪 / 与开发任务对齐 / 大批量节点修改 / 跨会话延续。
>
> 在 trade-ai 项目积累，约 2 周内完成 12 屏 + 30+ 子页面 + 多次重做迭代。

---

## 适用场景

- 用 AI 做 5 屏以上的中等到大型 UI 设计
- 需要跟开发任务（GitHub Issue / 后端 API）持续对齐
- 设计要多轮迭代（v1 → v5 这种）
- 需要在多次会话之间延续，不能每次重新开始

## 核心约束

Pencil MCP 关键限制（这些决定了工作流形状）：

| 限制 | 后果 |
|---|---|
| `batch_design` 单次最多 25 ops | 大批量改动必须切片 |
| `batch_get` `readDepth>3` 容易爆 token | 探索时分层读 |
| `patterns:[{type:"text"}]` 全量 dump 经常超限 | 走 `python3 -c "open(...).read()[A:B]"` 切片 |
| 节点 id 是隐式真理 | 写下来，否则下次找不到 |
| `D` (delete) / `U` (update) 操作可能延迟落盘 | 用户需要 Cmd+S，开发者用 `git status` 验 |
| `text` 节点没 `gap` 属性 | batch 里写错全 rollback，多想一秒 |

---

## 工作流分 5 段

### 1. 起步：建总览导航 frame

**最重要的第一步**。每次新会话只要「当前进度」一查就清。

在画布右侧（如 x=4700, y=0）画一个「📍 操作流程总览」frame：

- **4 列网格**对应 4 大业务区（A 入口 / B 看板 / C 主框架 / D 通用子页）
- 每列若干「卡片」对应一个屏 / 子页面：
  - 标题：`✅ ⏳ 🛠` 状态 emoji + 屏名
  - 副标题：入口 → 出口流向（1-2 行）
- 顶部 legend：`✅ 已完成 / ⏳ 部分完成 / 🛠 待开发`

```
┌──────────────────────────────────────────────────────────┐
│ 📍 操作流程总览（12 屏 + 子页面挂接图）                          │
├─────────┬─────────┬─────────┬───────────────────────────┤
│ A 入口流  │ B 看板  │ C 主框架  │ D 通用子页 / 跨屏              │
│ 屏 1     │ 屏 2    │ 屏 4     │ 屏 3 / 字段覆盖 modal /       │
│ sp1/2/3  │ sm1/2/3 │ 屏 5/6/7/8│ panel / 屏 16 / 状态组件 ...│
└─────────┴─────────┴─────────┴───────────────────────────┘
```

**为什么必要**：

1. 屏数一旦 > 5 个，靠记忆和滚动找会失控
2. 卡片描述「入口/出口」逼迫你想清楚跨屏跳转
3. 跟 GitHub Issue 双向同步，一份在 .pen 一份在 issue body

---

### 2. 主屏 + 流程说明 + 缺口子页面三件套

每个主屏（如「屏 5 v5 基本情况」）旁边放两个 frame：

**「流程说明」frame**（580px 窄）— 5 个色块：

```
inBox       入口（蓝边）   屏 4 nav「基本情况」
zonesBox    内部分区（蓝边） 6 区域：Hero / 详情 / 背景 / 历程 / 办事处 / 社媒
hasBox      已有子页面（绿边）字段覆盖 modal / 平台补全
gapBox      缺口子页面（黄边）平台补全候选 / 里程碑编辑 / 资信上传
outBox      出口（蓝边）    保存 / nav 切换其他 tab
```

**「缺口子页面」frame**（≥1340px 横）— 网格列出每个 modal/menu/panel 的设计草图

色块约定（用整个项目一致的颜色）：
- 蓝边 `#0075DE40` = 蓝白边框入/出口
- 绿底 `#ECFDF5` 绿边 `#10B98140` = 已有
- 黄底 `#FFF8E5` 黄边 `#9A670040` = 缺口
- 紫底 `#FAF7FF` 紫边 `#7C3AED40` = 思路对比

**为什么必要**：

1. 让"我以为做完了"和"实际还差什么"在同一屏可见
2. 子页面（modal / menu）在主屏没空间放，需要专门区域承载
3. 跨屏共享的 modal（字段覆盖 / 上传）只画一份，标注复用关系

---

### 3. GitHub Issue 同步：双向 tracking

建一个 tracking issue（如 #176 「屏 1-N 操作流程总览」），body 用 4 大区 checkbox：

```markdown
### A · 入口流（屏 1 + 子页面）
- [x] 屏 1 — 调研列表
- [x] sp1 — 新建调研 modal（#163）
- [ ] sp2 — 批量调研 6 步向导

### B · 批量看板（屏 2 + 子页面）
...

## 入口/出口流向（关键挂接）
| 来源 | 动作 | 目标 |
|---|---|---|
| 屏 1 | ⊕ 新建 | sp1 → submit → 屏 4(loading) |
...
```

**两边都要更新**：每完成一屏 →
1. 在 .pen 总览 frame 改 `✅`
2. 在 issue body 勾掉 checkbox + comment「设计稿同步 yyyy-mm-dd」

**为什么必要**：

1. 跨会话延续 — 新会话第一件事就是「看 issue + 总览图」找当前进度
2. 设计与开发共用同一份索引，开发实施时不会漏屏
3. 历史 comment 形成一手时间线（v5 重做、屏 X 删除等决策）

---

### 4. 大批量节点修改的工作流（如全局中文化）

当需要扫一遍全部 text 节点改一类内容（中文化、规范化、术语统一）：

#### Step A — Dump 全量 text 节点

```javascript
batch_get({patterns: [{type: "text"}], readDepth: 1})
```

Pencil 通常会返回「too large」错误并存到 `.../tool-results/xxx.txt`。

#### Step B — Python 切片读 + 程序化筛选

```python
import json, re
nodes = json.loads(open('xxx.txt').read())
hits = {}
for n in nodes:
    nid = n.get('id', '')
    c = n.get('content', '')
    matched = []
    for kw in ['Designer', 'Buyer', 'PetSmart', ...]:
        if kw in c:
            matched.append(kw)
    if matched:
        hits[nid] = (matched, c[:120])
```

输出：`{node_id: ([matched_keywords], snippet)}` 清单。

#### Step C — 把清单交回用户决议规则

不要 AI 自己决定全改，而是先列分类：

```
A 区 KP 职位词（约 10 个）— 全改？英文+中文括注？
B 区 国际品牌（约 15 个）— 全改？只改高频？
C 区 不常见企业名（保留原文不动）— 同意？
D 区 业务术语（fork / KP / YID 等）— 加注？
E 区 UI 通用 / 工具名 — 不动
```

用户拍板规则后再走 Step D。

#### Step D — 分批 `batch_design` U() ops（≤25/批）

```javascript
batch_design({operations: `
U("qrWbV", {content: "Designer 设计师 · 12"})
U("Jsqp9", {content: "+ Buyer 采购员 · 23"})
... // 22 条
`})
```

#### Step E — 漏网 grep + 第二轮

第一轮难全覆盖。完成后再 grep 一遍同样的关键词，找漏的（可能在长文本块、列表项里）。每批之间隔几秒避免 server 拒绝。

**为什么必要**：

1. AI 单 prompt 改不动百级节点 — 必须切片
2. 决议规则前置 — AI 自己决定常会改错（如把常见高管缩写 CEO/COO 也翻译了）
3. 程序化筛选 — 比眼看快一个数量级，且 reproducible

---

### 5. 跨会话延续：保留状态的 3 个手段

#### a. Memory file（auto memory）

记关键决策、节点 id、规范，比如：
> `屏 5 v5 = WMUnT, 字段覆盖 modal = Ldhdx（屏 5/8 共用）, 总览图 = RQoJT @ x=4700,y=0`

> `中文化约定：业务岗英文+中文（Designer 设计师），常见高管缩写不译（C-level/VP/GM/CEO/COO），不常见企业名保留原文`

#### b. Pencil 文件本身的 frame 标签

每个屏顶部 `labelW` frame 写完整描述：
```
"屏 5 — 基本信息（v5：Hero+基本+公司背景+发展历程+办事处+社媒）+ 字段覆盖 modal"
```

下次打开靠搜索 frame name 直接定位。

#### c. GitHub Issue body + comment

issue body 每隔几次更新做版本快照（设计稿现状、节点清单、改动决策）。
新会话先 `gh issue view N --json body` 拿状态。

---

## 反模式 / 踩过的坑

### ❌ 直接全文覆盖大文档

`Write` 工具替换整个 .pen 文件 → 会丢节点 id。永远用 `batch_design U/D/I` 增量改。

### ❌ AI 自作主张决定中文化规则

> 我把 `C-level → 高管层（C-level）` 全改了，用户说「常见高管缩写不译」—— 又得回滚一遍。

教训：**用户拍板规则前不要批量执行**，列清单 → 用户打勾 → 才改。

### ❌ 旧版本设计稿不删，靠记忆排查

屏 12/13/14 是「基本情况」的 v2/v4/v4-edit 三个迭代版本，全部已被屏 5 v5 覆盖，但旧版长期留在文件里造成「这个还有用吗」的反复确认。

教训：**v5 一旦稳定就在主分支删旧版**，不留备份。git 是备份，文件不是。

### ❌ git commit 不带 pathspec

yicha-workspace 仓库有 100+ 无关 staged 文件，`git commit -m "..."` 会把它们全部带进 .pen 的 commit。

正确做法：`git commit -m "..." -- wiki/designs/foo.pen` 用 pathspec 隔离。

### ❌ batch_design 里写出错的 schema

text 节点没 `gap` 属性，但 frame 有。混用了 → 整批 rollback。教训：先 batch_get 一个目标节点看 schema，再写 ops。

### ❌ 节点 id 不写下来

会话切到 compact 之后，原本「屏 5 v5 = WMUnT」记不清了。每次现查 5 分钟。

教训：**前 10 个最重要的节点 id 写到 memory file 里**。

---

## 复用 checklist（开新项目用）

```
[ ] 在画布右侧建「📍 操作流程总览」frame，4 列网格（按业务区分）
[ ] 每个主屏建 3 个伴生 frame：流程说明 / 缺口 / 思路对比（如有迭代）
[ ] 在 GitHub 建 1 个 tracking issue，body 用 checkbox 大纲对应总览图
[ ] memory file 记 10 个关键节点 id + 设计决策
[ ] 大批量改动用 python 切片 dump → 列规则给用户拍板 → 分批 25 ops
[ ] commit 用 pathspec 隔离，独立 push
[ ] 旧版本一稳定就删，靠 git 不靠文件
```

---

## 关联

- Pencil MCP 文档：见 MCP 服务的 `get_guidelines`
- 实际项目：trade-ai 仓库 commit `c5224bf` / `6ce48c6`（屏整理 + 中文化）
- GitHub Issue 模式：trade-ai #176（活的 tracking issue 范式）
