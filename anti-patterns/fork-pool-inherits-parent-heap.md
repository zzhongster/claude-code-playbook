# 反模式：在父进程装满大对象之后再 fork 进程池

## 一句话结论

Linux 上 `ProcessPoolExecutor`/`multiprocessing.Pool` 默认 fork：子进程"共享"父堆只是写时复制的幻觉——Python 一碰引用计数就写页，36 个 worker 会各自把父进程的 GB 级 dict 复制一遍。归一化这种**无状态**的工作要用 `spawn` 上下文（worker 自己加载词典），而且要把它排在父进程还小的阶段；真正需要共享只读大数据的池，用 Arrow/numpy 缓冲区（单块、无引用计数写）+ `gc.freeze()`，并在 fork 前 `del` 掉不再需要的临时对象。

## 场景

- 流水线后段（S4）先把整片名称装进 Python dict，再调用一个"顺手"的归一化进程池
- 昨天没 cgroup 时侥幸跑过（66 GB），今天在 100 GB cgroup 下被 OOM-kill

## 踩坑经历

2026-08-30 企业样本去重离线试跑：`run_s4_cn` 在装入 90 万名称的 names/ev 字典后调用 `build_orbis_norm`，后者用 36 worker 的 fork 池归一 1.2 亿行 orbis 名。cgroup 统计 rss 104 GB，被杀的 worker 单进程 anon-rss 7.3 GB——worker 本身只处理 2 万个名字一批，7 GB 全是从父进程 COW 复制来的。改成 `mp_context=get_context("spawn")` 并把 orbis 归一提前到 prelink 阶段后，同一阶段 47 GB；随后把 S2/S4 的整片 dict 改成 Arrow 列式 + numpy 并查集，6.25% 样本峰值从 70 GB 降到 9.7 GB，且产物与基线逐行等价。

顺带两个坑：自定义入口脚本用 spawn 必须带 `if __name__ == "__main__"` 守卫，否则每个 worker 重跑整条流水线互抢文件锁；`ls -t dir1 dir2` 不是全局按时间排序，按阶段标注内存采样要用产物 mtime 划区间。

## 正确做法

- 无状态 worker → spawn；共享只读大数据 → fork + Arrow/numpy 缓冲 + `gc.freeze()`，fork 前 `del` + `gc.collect()`
- 重的常数阶段（企业库归一）放在父进程最小的时候做，缓存按版本复用
- 用 cgroup（`systemd-run --scope -p MemoryLimit`）跑，并按阶段采样 `memory.usage_in_bytes`，两个样本规模做斜率外推，别只看一个点
