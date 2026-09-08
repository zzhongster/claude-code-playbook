# 盘点并行 AI 会话留下的孤悬分支：cherry 判等价、merge-tree 干跑判冲突

## 一句话结论

多个 Claude 会话（桌面端会自动开 `claude/<随机名>` 分支）并行几天后，仓库里会留下一批**领先 main 的分支**：有的内容早已以另一个 hash 进了 main，有的是**做完、测过、没归档、没卡、没人知道**的真修复。三条 git 命令能在不碰工作区的前提下把它们分清楚：`git cherry`（补丁等价）、`git merge-tree --write-tree`（冲突干跑）、`git merge-base --is-ancestor`（祖先关系）。

## 场景

- 团队/个人用多个 CC 会话并行（worktree、桌面端、不同机器）
- `git branch --no-merged main` 列出一堆分支，看提交信息分不清哪些已经进了 main
- 主目录被另一个活跃会话占着，不能 checkout 任何东西来试合并

## 详细说明

三步，全部只读：

1. **补丁等价**：`git cherry main <branch>`。前缀 `-` 表示该提交的补丁（patch-id）已在 main 上——哪怕 hash 不同、被 squash/rebase 过；`+` 表示 main 没有等价补丁。全 `-` 的分支直接删。
2. **冲突干跑**：`git merge-tree --write-tree main <branch>`（git ≥ 2.38）。退出码 1 + `CONFLICT (content): … <file>` 就是真合并会撞的文件，工作区一个字节不动。用它决定"能不能直接 cherry-pick"，以及冲突在哪个文件、什么形态（两块各自追加的 add/add 最常见，并存即可）。
3. **祖先关系**：`git merge-base --is-ancestor <sha> main`，核"这个具体提交进 main 了没"。

`+` 但 merge-tree 说 main 上功能已存在（重写形态合入）的分支，看 `git diff main <branch> -- <它改的文件>`：只剩 main 领先的删除行就是已合入。

拿回一条孤悬修复的做法：**在新 worktree 里 `git cherry-pick <sha>`**（不 merge 整条分支——分支基点老，会把无关历史带进来），解冲突，跑全量，提交说明末尾加一段 cherry-pick 备注（来源 sha、基点、冲突文件与处置），再走正常评审/部署。原分支等 ff 之后再删。

## 数据支撑

trade-agent-data，2026-09-08 盘点 5 条领先 main 的分支：

| 分支 | `git cherry` | 判定 | 处置 |
|---|---|---|---|
| `claude/nice-chaum-1c6d4e`（2 ahead） | `- -` | 全等价已在 main | 删 |
| `fix/skill-direct-mcp`（2 ahead） | `- -` | 同上 | 删 |
| `fix/supplier-dedup-merge`（2 ahead） | `+ +` | 功能以重写形态进了 main（函数名在 main 上能 grep 到） | 删 |
| `claude/sad-khorana-6d9550`（1 ahead，3 天前） | `+` | **真孤悬**：192 行修复 + 7 条 RED 先行测试 + 契约文档改动，无卡无归档；merge-tree 报 1 个文件冲突（文件尾两块各自追加） | cherry-pick 进 worktree，两块并存，全量 1711 passed，当天部署 |
| `feat/trade-report-skill`（2 ahead，2 周前） | `- +` | 一半进了，另一半（138 行）没进 | 留给人决定形态 |

另有 13 条 `--merged main` 的 feat/* 分支和一个停在孤悬提交上的 detached worktree 一并清掉。

那条真孤悬的修复，从做完到被发现隔了 3 天——桌面端会话不写归档、不开卡、分支名是随机词，`git log` 里看不出它是什么。**盘点时先跑 `--no-merged` 而不是先看 Linear**，是这次能捞到它的原因。

## 适用范围

- 适用于：任何多会话/多人并行的仓库，尤其是有 AI 会话自动建分支的
- 不适用于：分支本身就是长期特性分支（不该被判"孤悬"）；补丁被大幅重写后 `git cherry` 会给 `+`，要人看 diff 兜底

## 相关

- [git-flow-for-solo-ai-first-dev](git-flow-for-solo-ai-first-dev.md)
- [handover-doc-trio-parallel-archaeology](handover-doc-trio-parallel-archaeology.md)（交接文档的考古；本条是分支层的考古）
- [concurrent-claims-on-serial-resource](../anti-patterns/concurrent-claims-on-serial-resource.md)
