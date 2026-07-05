# 整分支终审抓涌现缺陷（Whole-Branch Review Catches Emergent Defects）

> subagent-driven 开发里，per-task 评审天然看不到"跨任务 / 规模涌现"的缺陷——每个任务的评审只看自己那块 diff，且 fixture 都是绿的。**分支合并前，必须再做一次「整分支 + 最强模型」的终审**。这一步不是走过场，它抓的是另一类 bug。

## 核心机制：为什么 per-task 评审必然漏

subagent-driven-development 每个任务派一个 implementer + 一个 task-reviewer。task-reviewer 只拿到**本任务的 diff** 和**本任务的 spec**。这是它的强项（聚焦），也是它的结构性盲区：

- **跨任务缺陷**：任务 N 的改动破坏了任务 M 的假设，但两份 diff 分开看都"对"。
- **规模涌现缺陷**：单元测试用小 fixture 全绿，缺陷只在**真实全量数据**下才出现。
- **假绿**：任务的测试 fixture 恰好把新代码路径关掉了（如 `hs()→None` 让派生逻辑不触发），测试绿但没测到真行为。

这三类，每个 per-task 评审看自己那块都通不过它的"雷达"。只有把整分支放到一起、用最能干的模型（Opus 级）审，才照得到。

## 两个实战案例（trade-agent-data，贸易数据 MCP）

| 薄片 | per-task 全绿，终审(Opus)抓到的 Important | 缺陷类型 | 根因 |
|------|------------------------------------------|---------|------|
| **slice-4 双模型** | 派生 HS 的 `should` 子句是**全局**的 → 把货描模型国家也 broadening（保温杯 us 81,592→103,377，水杯 5.1×）。所有 per-task 评审都放过了。 | 跨任务 + 假绿 | 回归测试的 fixture 让 `hs()` 返 None，**新代码路径根本没被触发**；新可选行为 × 旧数据形态的交叉没测 |
| **slice-5 词典扩容** | 反向索引 first-wins 把上游 3 条脏译词放大成错召回（`eggplant` 含 `cà chua`=番茄；`plums` 含 `prunes` 遮蔽真西梅干 0813；`cat` 含 `猫摆件`）。 | 规模涌现 | 词典从 2 品→1000 品，脏数据只在全量下出现；小 fixture 测不到 |

两次，per-task 评审都判"通过"，都是终审在**整分支 diff + 真实数据抽样**里挖出来的。一次是巧合，两次是规律。

## 落地做法

1. **终审是流程的固定一环，不是可选项**。所有任务完成后、合并前，必做。
2. **用最强模型**。per-task 评审可以按 diff 大小降级用便宜模型，终审不行——它要的是判断力，见 anti-pattern [cheap-model-judgment](../anti-patterns/cheap-model-judgment.md)。
3. **喂它整分支 diff（一个文件），别喂历史**。`review-package MERGE_BASE HEAD` 生成含 commit 列表 + stat + 完整 U10 diff 的单文件，评审 Read 一次即可，不进主循环上下文。
4. **明确要它抽样真实数据 / 全量产物**。大数据文件（如 1000 行词典）别逐行读，让它抽样统计"有无空值 / 脏值 / 越界"。规模涌现缺陷只有这样才看得到。
5. **把发现变成自动守卫**。终审抓到的每类缺陷，补一条能自动抓它的东西：
   - slice-4 → 回归测试覆盖"新行为开启 × 旧数据形态"交叉，禁止用假 fixture 关掉新路径。
   - slice-5 → `build_keywords.py` 加构建期"遮蔽 / 错 HS"告警 + golden 回归天花板（只允许已知可接受多义）。
   
   否则同类缺陷下次还会从别的入口回来。

## 对应关系

- 与 [code-audit-checklist](code-audit-checklist.md)（何时触发审计）互补：那份讲"里程碑后主动审"，这份讲"per-task 审之外，为什么终审不可省 + 它专抓哪类"。
- 反面教材：[cheap-model-judgment](../anti-patterns/cheap-model-judgment.md)——终审用便宜模型 = 白审。
- 假绿的具体形态见 [round-trip-projection-loses-fields](../anti-patterns/round-trip-projection-loses-fields.md) 一类"测试没测到真路径"。
