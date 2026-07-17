# Claude Code 长任务三件套 — nohup 脱离 + 每步 checkpoint + 断点续跑

> 来源：trade-ai 深度报告验证跑实战（2026-07-16，同一任务被杀 2 次后总结）
> 症状：Claude Code 的后台 Bash（run_in_background）跑 ~30 分钟级长任务会被 harness 静默终止

## 踩坑过程

跑 30 分钟的深度报告验证（10 步 LLM 流水线，成本 ¥10+/次）：

1. 第一次用 `run_in_background: true` 后台跑 → **~93 分钟处被 killed**（进程无 traceback 静默死亡，harness 通知 status: killed）
2. 单任务重启再跑 → **~29 分钟处又被 killed**，且步骤产物只在内存里——两次合计 ¥20+ 白烧
3. 结论：Claude Code 对后台任务有自己的生命周期管理，长任务不能托付给 `run_in_background`

## 三件套解法

### 1. nohup + disown 完全脱离
```bash
nohup python3 -u runner.py > run.log 2>&1 & disown
```
进程脱离 harness 任务树，后台管控杀不到。实测脱离后同任务 3 次跑完无一被杀。

### 2. 每步 checkpoint 落盘
长流水线每完成一个步骤立即把产物写盘（JSON checkpoint），别等全部跑完：
```python
ckpt["steps"][step_id] = {"output": ..., "event": ...}
json.dump(ckpt, open(CKPT_PATH, "w"), ensure_ascii=False)
```

### 3. 守护循环断点续跑
```bash
nohup bash -c 'for i in 1 2 3; do python3 -u runner.py && break; echo "崩溃，续跑 $i"; sleep 5; done' >> run.log 2>&1 & disown
```
runner 启动时先读 checkpoint，已完成步骤直接跳过——崩溃只损失当前一步的钱。

### 4. 进度感知：监控器盯日志（不是盯进程）
脱离后 harness 拿不到完成通知，用 Monitor 持续 tail 日志文件的关键行：
```
tail -f run.log | grep --line-buffered -E "step_done|完成|Traceback|致命"
```
每步完成自动收到通知，异常第一时间可见。

## 额外教训

- **杀掉的可能只是外层 shell**：`ps` 查到"活着"的可能是监控的 tail，python 早死了——判断进程存亡看目标进程名，不看任务状态
- **产物落盘时机**：markdown/最终产物如果只在"全部完成后"写盘，中途死亡=全部丢失。事件流边跑边 flush 到 jsonl，最终产物尽早写
- 环境注意：CLAUDE.md 若有 Python 版本约束（如避开 3.14），长任务前先确认——原生扩展在新版本上的无声段错误与 harness 杀进程症状难以区分
