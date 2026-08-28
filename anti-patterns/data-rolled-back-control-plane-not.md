# 反模式：回滚了业务数据，没回滚控制面状态

## 一句话结论

一次写入如果同时更新了"数据行"和"描述这些数据的状态"（分片版本、水位、last_run_id、缓存 manifest），回滚就必须两者一起回；只回数据不回状态，系统会认为"新版本已经生效"，下一次增量据此跳过本该重做的工作——数据回到旧版，标签还是新版，而且不会报错。

## 场景

- 结果表按 run 写入，另有一张 `shard_state(cc, lib_version, pipeline_version, last_run_id)` 记录"这个分片当前是哪个版本算出来的"
- 增量任务用 `shard_state.lib_version` 与当前企业库版本比较，决定要不要全分片重算
- 回滚实现只关注结果表（pre-image 恢复行、删除新行）

## 踩坑经历

2026-08-29 企业样本去重实施计划第六轮评审。我刚在上一轮把"分片版本"从行级 `any_value(lib_version)` 改成独立的 `shard_state` 表（本身是对的），`writeback` 成功后写入 L2。评审指出：L2 全量重算后回滚到 L1，结果行已经是 L1，`shard_state` 仍是 L2；下一次企业库仍是 L2 时，增量判定 `lib_changed=false`，**必须执行的企业库重关联被静默跳过**。

同一模式在别处也见过：Kafka offset 提交了但下游落库回滚；缓存 manifest 写了新版本但派生文件重建失败；`last_success_at` 更新了但产物被删。

## 正确做法

- **状态也要 pre-image**：写入前把 `shard_state` 当前行存入 `shard_state_preimage(run_id PK, existed, ...)`（幂等：已存在则不覆盖，重试不会用新状态冲掉真正的旧状态）
- **回滚按条件恢复**：
  - `shard_state.last_run_id == 被回滚 run` → 用 pre-image 恢复；首次 run（`existed=false`）→ 删除状态行
  - 已有后续 run 写过该分片 → **保留后续状态**，返回值明确标 `shard_state: kept`，不能覆盖别人的进度
- 测试至少覆盖：L1 → L2 → 回滚 L2 后 `shard_state.lib_version == L1`；写 L3 后回滚 L1，状态仍是 L3
- 更一般的判断法：列出一次写入触碰的**所有**表/文件/外部系统，每一个都问"回滚时它怎么办"；答不上来的就是漏洞

## 识别信号

- `rollback()` 只出现结果表的表名
- 状态表的写入在 `writeback` 末尾"顺手"加上，没有对应的读回路径
- 增量/调度逻辑依赖某个"当前版本"字段，而这个字段没有版本历史
