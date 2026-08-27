# 反模式：清掉大占用者后立刻采样，把「饥饿服务的积压反弹」误读成新元凶（外加一次「怪工具不准」的误判）

## 一句话结论

杀掉一个长期占满 CPU 的进程之后**立刻**采样，会看到一批系统后台服务集体冲高——那不是新元凶，是被饿了几十小时的服务在追赶积压；而当你发现「几分钟后它们又不见了」时，第一反应若是「**这个工具的读数不准**」，那多半是你没读 man page。

## 场景

- 刚 kill 掉一个大占用进程，想确认效果，马上又跑了一次 `ps` / `top`
- 排名里冒出一批你没动过的系统进程（Spotlight、云盘同步、Siri 建议一类）
- 过几分钟再看，它们又消失了，于是怀疑测量工具在骗你

## 详细说明

### 现象

2026-08-28，杀掉一棵吃 823% CPU 的孤儿 headless Chrome（详见 [orphaned-headless-browser-burns-cpu](orphaned-headless-browser-burns-cpu.md)）后**立即**采样：

```
$ ps -Ao pid,pcpu,pmem,etime,comm -r | head
 1393  99.8   com.apple.Virtualization.VirtualMachine   ← Docker VM
  364  97.7   Metadata.framework/.../mds                ← Spotlight 索引
  701  96.2   FileProvider.framework/.../fileproviderd  ← 云盘同步
98359  50.4   duetexpertd                               ← 系统智能
 6239  48.2   suggestd                                  ← CoreSuggestions
```

几分钟后用 `top` 二次采样取瞬时值，**除 Docker VM 外全部沉底，不在前 12 名**。

### ❌ 我当场给出的解释（错的）

> 「`ps` 的 %CPU 是**进程生命周期累计均值**，对跑了 46 天的进程会虚高；`top` 才是瞬时值。」

这个解释很顺口，甚至有一半是对的（`top` 确实该用二次采样），而且它导向的行动建议也碰巧正确（别去杀那几个系统进程）。**但归因是假的。**

### ✅ `man ps` 当场证伪

```
%cpu   The CPU utilization of the process; this is a decaying average
       over up to a minute of previous (real) time.
```

macOS 的 `ps %cpu` 是**过去最多一分钟的衰减平均**。生命周期累计那是 **Linux** 的 `ps` —— 我把两个平台的语义搞混了，然后用错的那个去解释一个 macOS 上的现象。

也就是说：**`ps` 没有说谎。那一分钟里，`mds` 和 `fileproviderd` 是真的在烧 CPU。**

### 真正的原因：饥饿服务的积压反弹

那四个进程的共同性质是**可延后的后台工作**——文件索引、云盘同步、使用习惯建模。CPU 被占满的 31 小时里，调度器一直压着它们；**释放的那一刻它们同时扑上来追赶积压**。

「四个互不相关的服务同时冲高、又同时在几分钟内回落」这个形状本身就是证据：指向一个共同的外部原因（CPU 突然可用），而不是四个各自碰巧忙起来的巧合。

### 两条判据

1. **清理后的第一勺不是稳态。** 要测效果，等 3–5 分钟，或者直接用 CPU 时间增量法（不受采样口径影响）：

   ```bash
   t1=$(ps -o time= -p <PID>); top -l 4 -s 5 -n 0 >/dev/null; t2=$(ps -o time= -p <PID>)
   echo "$t1 -> $t2"    # CPU 时间增量 / 墙钟增量 ≈ 占了几个核心
   ```

   这一招在本次排查里立了大功：它证明 Docker VM 的 102% 是**真的持续在烧**（16.05 秒 CPU / 约 15-20 秒墙钟），而不是和 `mds` 一样的反弹假象。**同一份榜单上，有的行是假象有的行是真的**——所以判据必须逐行验，不能整榜一起信或一起否。

2. **「工具读数不准」是一个需要证据的断言，不是一个解释。** 两个时刻的观测不一致，最常见的原因是**这两个时刻之间真的发生了变化**，而不是工具坏了。怀疑工具之前先花 10 秒读 man page——本次从起疑到证伪，成本就是一条 `man ps | grep`。

## 数据支撑

| 观测时刻 | mds(364) | fileproviderd(701) | Docker VM(1393) |
|---|---|---|---|
| 杀掉 Chrome 后 0 分钟（ps） | 97.7% | 96.2% | 99.8% |
| 约 5 分钟后（top 瞬时） | 未进前 12 | 未进前 12 | **102.0%** |
| 结论 | 反弹，假象 | 反弹，假象 | **真的在烧** |

## 适用范围

- 适用于：任何「释放了大量资源之后立刻测量效果」的场合——CPU、内存、磁盘 IO、数据库连接池同理
- 适用于：跨平台工具的语义差异（`ps`、`top`、`free`、`df` 在 macOS / Linux / BSD 上都有口径差别）
- 不适用于：持续几十分钟不回落的高占用（那就不是反弹，去查真实原因）

## 相关

- [孤儿 headless 浏览器空转烧 CPU](orphaned-headless-browser-burns-cpu.md)——本次被清掉的那个元凶
- [装带本地 daemon 的 Skill 不配套关闭手段](skill-with-local-daemon.md)——元凶的来源类别
