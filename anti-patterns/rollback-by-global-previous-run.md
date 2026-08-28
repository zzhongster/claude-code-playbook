# 反模式：回滚时为所有受影响 id 选同一个"上一次 run"，增量场景下部分 id 被误删

## 一句话结论

结果表按 id 覆盖、历史表按 run 追加的模式下，"回滚 run R" 不是"恢复到 run R-1"——每个 id 的上一版本可能来自不同的 run。回滚必须**逐 id** 找 `written_at < R` 的最新历史版本，没有更早版本的 id 才删除；历史表本身还要对重试幂等。

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

```sql
-- 逐 id：每个 id 取早于本 run 写入的最新版本
INSERT INTO res (cols)   -- 会话内先 SET enable_unique_key_partial_update = true
SELECT h.cols FROM hist h
JOIN (SELECT p.id, max(p.written_at) w FROM hist p
      JOIN hist cur ON cur.id = p.id AND cur.run_id = :run
      WHERE p.written_at < cur.written_at GROUP BY p.id) m
  ON m.id = h.id AND m.w = h.written_at;
DELETE FROM res WHERE run_id = :run;   -- 剩下仍标记为本 run 的 = 没有更早版本的 id
```

- 历史表 `UNIQUE KEY(run_id, id)`（merge-on-write）+ Stream Load **确定性 label**（`hist-{run_id}`）：重试要么被拒要么覆盖同一行，永远只有一个版本
- 测试必须包含"混合来源"用例：r1 写 {A,B}，r2 写 {A,B}，r3 只写 {B,C}，回滚 r3 后 A 仍是 r2、B 回到 r2、C 被删

## 识别信号

- 回滚代码里有 `ORDER BY ... LIMIT 1` 选出一个 run 再套所有 id
- 历史表是 DUPLICATE/append 模型且写入用随机 label
- 回滚测试只覆盖"全量 run 之间"的回滚
