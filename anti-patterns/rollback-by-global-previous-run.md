# 反模式：回滚时为所有受影响 id 选同一个"上一次 run"，增量场景下部分 id 被误删

## 一句话结论

结果表按 id 覆盖、历史表按 run 追加的模式下，"回滚 run R" 既不是"恢复到 run R-1"，也不是"逐 id 取最近的历史版本"（后者会复活已被回滚的版本）。唯一正确的依据是**写入前的 pre-image**：每次写入前存下受影响 id 的实际旧值，回滚只恢复它，被后续 run 覆盖过的 run 拒绝回滚。

## 场景

- 当前态表 `UNIQUE KEY(id)`，每次 run 用 partial update 覆盖一部分 id
- 增量运行只重算"受影响簇"，一次 run 触碰的 id 之前可能分别在 r1、r2、r3 写过
- 需要"按 run_id 整批回滚"

## 踩坑经历

2026-08-29 实施计划评审。原 `rollback`：

```python
prev = "SELECT run_id FROM hist WHERE id IN (本 run 的 id) GROUP BY run_id ORDER BY max(written_at) DESC LIMIT 1"
INSERT ... SELECT FROM hist WHERE run_id = prev AND id IN (...)
DELETE FROM res WHERE run_id = 本 run     # 上一 run 没覆盖到的 id 在这里被删掉
```

增量 run r3 改了 id A（上次在 r2 写）和 id B（上次在 r1 写）。全局 `prev = r2`，只恢复了 A；B 仍带着 r3 的 run_id，被最后一句 DELETE 删掉——**数据丢失，且没有报错**。

另一处：历史表是 `DUPLICATE KEY(run_id, id)`，Stream Load 超时重试会写出第二份同 run 版本，回滚时 `max(written_at)` 取到的是重试那份而不是逻辑上的"上一版"。

## 正确做法

**第一版修正（仍然错）**："逐 id 取历史表里 `written_at` 早于本 run 的最新版本"。它修好了增量混合来源的问题，但引入了另一个：r3 → 回滚到 r2 → r4 → 回滚 r4 时，历史表里时间最近的是 r3，于是把已经回滚掉的 r3 复活了。历史表记录的是"写过什么"，不是"写之前是什么"。

**真正的做法：pre-image。** 每次写入前，把本 run 触及的每个 id 在结果表里的**实际旧值**（含"此前不存在"）存一份，回滚只恢复这份：

```sql
-- 写入前（history 已写、result 未动）
INSERT INTO preimage (run_id, id, existed, <value cols>, prev_run_id)
SELECT h.run_id, h.id, r.id IS NOT NULL, <r.value cols>, r.run_id
FROM history h LEFT JOIN result r ON r.id = h.id
WHERE h.run_id = :run AND h.id NOT IN (SELECT id FROM preimage WHERE run_id = :run);   -- 重试不覆盖真正的旧值

-- 回滚
-- 0) 拒绝：本 run 触及的 id 已被后续 run 覆盖（result.run_id <> :run）→ 先回滚后续 run
-- 1) SET enable_unique_key_partial_update = true; INSERT INTO result (<cols>, run_id) SELECT <cols>, prev_run_id FROM preimage WHERE run_id = :run AND existed
-- 2) DELETE FROM result WHERE run_id = :run           -- 剩下的 = existed=false 的 id
-- 3) DELETE FROM preimage WHERE run_id = :run         -- 已消费，防二次回滚
```

- pre-image 表 `UNIQUE KEY(run_id, id)`；history 表也改 `UNIQUE KEY(run_id, id)` + Stream Load 确定性 label，只做审计与增量对比，不参与回滚
- 测试必须覆盖：r1 {1,2} → r2 {1,2} → r3 {2,3} → 回滚 r3 → r4 {2} → 回滚 r4，断言 id 2 回到 **r2**；以及"r5 覆盖了 r2 的 id 后回滚 r2 被拒绝"

## 识别信号

- 回滚代码里有 `ORDER BY ... LIMIT 1` 选出一个 run 再套所有 id，或按 `max(written_at)` 从历史表找"上一版"
- 历史表是 DUPLICATE/append 模型且写入用随机 label
- 回滚测试只覆盖"全量 run 之间"的回滚
