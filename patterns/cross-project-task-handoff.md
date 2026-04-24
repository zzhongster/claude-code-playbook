# 跨项目任务派发（Cross-Project Task Handoff）

## 一句话结论

> 不要让 Claude 跨项目目录直接动手，**派任务**：在对方仓库放一个标准化 markdown 任务文件，
> 让对方那边的 Claude 会话按文件执行。

## 场景

单人多项目工作流里非常常见：

- 项目 A（如官网）发现项目 B（如内容生成器）需要修一些东西
- 自然反应：在 A 的 Claude 会话里 `cd ../B` 直接动手
- 后果：
  - A 这边 Claude 不熟悉 B 的代码结构和约定，容易踩坑
  - A 的 CLAUDE.md / 上下文不适用于 B
  - 改动散落在 A 的 commit 里，B 仓库 git log 没记录
  - 没有验收标准，"做完了"≠"做对了"

更稳的做法：A 派一个**结构化任务文件**给 B，由 B 那边新开 Claude 会话执行。

## 落地结构

### 收件方（callee）—— 设 `docs/tasks/` 收件箱

```
B 项目根/
├── CLAUDE.md                 # 顶部说明会话开始时 ls docs/tasks/
└── docs/tasks/
    ├── README.md             # 任务文件格式约定
    ├── <slug>.md             # 待办任务（pending / in_progress）
    └── done/<slug>.md        # 已完成归档
```

CLAUDE.md 加一节：

```markdown
## 跨项目任务收件箱

`docs/tasks/*.md` 是其他项目派来的任务。
**会话开始时先扫一眼：** `ls docs/tasks/*.md`，按 priority 处理 pending 的。
完成后 status=done，文件移到 `docs/tasks/done/`。
```

### 发件方（caller）—— 设 `docs/cross-project-tasks/sent/` 发件箱

发件箱的作用纯粹是**让 caller 仓库的 git log 留痕**——派出过什么、什么时候派的、当时为什么派。

```
A 项目根/
├── CLAUDE.md                 # 加一节"跨项目派任务"流程
└── docs/cross-project-tasks/sent/
    └── <slug>.md             # 与 callee 任务同名副本
```

CLAUDE.md 加一节：

```markdown
## 跨项目派任务（给 B）

发现 B 那边需要改动时：
1. 在 `<B 路径>/docs/tasks/<slug>.md` 写任务文件
2. 在本项目 `docs/cross-project-tasks/sent/<slug>.md` 留同名副本
3. 告诉用户「在 <B 路径> 开 Claude Code，处理 docs/tasks/<slug>.md」
```

## 任务文件标准模板

```markdown
---
id: <slug>
from: <发起项目>
to: <目标项目>
created: YYYY-MM-DD
status: pending           # pending → in_progress → done
priority: P0|P1|P2
---

# <标题>

## 来源
- 发起项目路径 / 分支 / commit

## 背景
为什么要做这件事，上下文。

## 待办
1. 步骤一（具体到文件路径、命令）
2. 步骤二

## 验收标准
- [ ] 可执行的检查命令（grep、curl、ssh ... | grep -c）
- [ ] 不要写"代码看起来对"这种主观判断

## 白名单 / 黑名单
不要碰的东西。明确边界，避免对方误清。

## 收尾
完成后做什么、产物部署到哪里、怎么回执。
```

## 为什么有效

1. **上下文自足**：任务文件本身就是完整 prompt，对方 Claude 不需要回看本对话
2. **验收可执行**：用 grep/curl 写验收，不是"看起来对"的主观判断
3. **双向追溯**：caller 仓库 git log 看派出过什么；callee 仓库看处理过什么
4. **白名单约束**：明确"不要动什么"，比明确"要动什么"更能防止对方误操作
   （LLM 倾向"过度修改"，写黑名单比写白名单更有效）
5. **责任清晰**：caller 负责定义"对错"（验收），callee 负责实现，不互相猜

## 何时该派任务，何时该自己上

| 情况 | 派任务 | 自己跨目录干 |
|------|-------|------------|
| 改动 < 5 分钟、不涉及 callee 业务逻辑 | | ✓ |
| 涉及 callee 的 CLAUDE.md 约定 / 模板 / 数据源 | ✓ | |
| 改动产物需要 callee 重新构建/同步/部署 | ✓ | |
| 需要跑 callee 的命令、用 callee 的虚拟环境 | ✓ | |
| 一次性纯文本改动（如改个 typo） | | ✓ |

## 反模式

### 跨目录直接动手 + commit 到 caller 仓库
git log 显示 caller 项目的 commit 改了 callee 仓库无关文件。半年后回看 callee 项目，
完全找不到这个改动是哪儿来的、为什么改的。

### 任务文件只写"待办"，不写验收
"清理一下文章里的过期数据"——对方 Claude 改了一部分就回报"完成"。结果只清理了 60%。
**验收必须是可跑的命令**，输出 0 或非 0 才能判定。

### 任务文件没写白名单
"清理所有教师人数"——对方 Claude 把所有数字（学费、面积、年份）一并清理了。
**显式列出"不要动什么"**比"要动什么"更能止损。

## 适用范围

- 适用于：单人多项目、Monorepo 拆分仓库、跨团队任务派发
- 不适用于：单仓库内任务（用 TodoWrite 就够）、需要实时来回沟通的探索性任务
- 进阶：如果跨项目任务非常频繁，可以考虑 GitHub Issue / Linear 等正式 issue tracker
  代替 markdown 文件（但本地 markdown 的好处是不依赖网络、git 直接追溯）

## 成熟度

**初步实践**：目前在 1 对项目（`shao` 官网 ↔ `geo` 内容生成器）验证过 1 次。
后续在更多项目对上跑通后再补充。

## 相关

- [Living CLAUDE.md](living-claude-md.md) —— 收发件箱要写进 CLAUDE.md 才会被读
- [Decision Records](decision-records.md) —— 任务文件本质是轻量级 ADR
