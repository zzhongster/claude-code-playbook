# 反模式：回滚用"按 run_id 删除"收尾，而恢复写入还没可见

## 一句话结论

在写入提交与可见分离的存储（Doris/StarRocks MOW 表、异步 publish 的 OLAP）上，"先 INSERT 恢复旧值，再 `DELETE … WHERE run_id = 本 run`"这种回滚会把**刚恢复的行**一起删掉：DELETE 扫描到的是尚未可见的旧版本，命中条件成立；随后的"结果表无本 run 行"校验又天然通过，pre-image 被消费——静默且不可恢复。DELETE 必须按 **id 集合**限定（只删 pre-image 标记为"此前不存在"的 id），让结果与可见性顺序无关。

## 场景

- 回滚两阶段：数据阶段用 pre-image 恢复（`existed=true` 的 id partial update 回旧值，`existed=false` 的 id 删除），随后校验"结果表中不再有 run_id = 本 run 的行"，通过后删 pre-image
- 恢复 INSERT 走一条连接（需要 `SET enable_unique_key_partial_update`），DELETE 走另一条新连接
- 出现"偶发一次、重跑不复现"的活库测试失败

## 踩坑经历

2026-08-29 企业样本去重 Task 14 回写/回滚。实现者报告活库测试 10 次里 1 次失败、无法复现，归因"读后写可见性"想加重试。评审指出真正的坏路径：`DELETE FROM result WHERE run_id='<run>'` 不带 id 限定，MOW 表 DELETE 是"扫描匹配行写删除标记"，若恢复行尚未 publish，它扫到的是旧版本并整体标删；接着 `left=0 / still=0` 校验通过，两份 pre-image 被删除。**加重试治不了它**，因为坏结果本身能通过校验。

## 正确做法

- 回滚 DELETE 写成 `DELETE FROM res WHERE run_id='<run>' AND id IN (SELECT id FROM preimage WHERE run_id='<run>' AND NOT existed)`——最坏情况退化为可重跑的校验失败，而不是删掉恢复行
- 校验阶段的 count 允许**有界**重试（区分 publish 延迟与真不一致），但这只是补充，不替代 id 限定
- 快照/pre-image 表要幂等：已非空则跳过写入，重试不会用"推回之后"的值覆盖"推回之前"的值
- 回滚之后不要复用同一 run_id：Stream Load label 与 run_meta 仍在，重跑会被 label 去重成 no-op，最后报一个指错方向的"行数不符"
