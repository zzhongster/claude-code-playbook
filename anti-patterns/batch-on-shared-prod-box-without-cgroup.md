# 反模式：在跑生产服务的机器上放批处理，只靠 nice / taskset / ulimit -v 兜底

## 一句话结论

`nice` 只管 CPU 调度优先级，`taskset` 只管核绑定，`ulimit -v` 是**每进程**的虚拟内存上限——三者加起来对"整棵进程树吃光物理内存"毫无约束。DuckDB 默认把 `memory_limit` 设成物理内存的 80%，再开一个 16 进程的 fork 池，机器会在几分钟内进入用户态卡死：TCP 端口还能连、ssh 握手被关、生产 HTTP 无响应，最后只能关机。共享机器上的批处理必须用 cgroup 卡整棵树，并给每个内存大户显式上限。

## 场景

- 算力机同时跑生产服务（trade-ai、Doris manager、nacos）
- 批处理脚本：`nohup nice -n 10 taskset -c 0-39 … ; ulimit -v 100G`
- DuckDB 多连接、multiprocessing fork 池，没人设 `memory_limit`

## 踩坑经历

2026-08-29 企业样本去重 CN-trial-b3 在 192.168.1.24（72 核 / 251 GB，跑生产）启动后约 7 分钟，ssh 报 `kex_exchange_identification: Connection closed by remote host`，`:8000` 5 秒无响应，ping 不通但 22/8000/6379 端口 TCP 可连——内核在、用户态死。用户最终关机。评审在前一小时其实已经指出：(a) DuckDB 从未设过 `memory_limit`，默认 ≈201 GiB > 100 GiB 的 ulimit，真到量是 malloc 失败或吃光内存，而不是落盘；(b) `-v` 是每进程上限，40 个 worker 各 100 G 等于没限；(c) `taskset -c 0-39` 在 HT 机器上覆盖了全部物理核，生产只剩超线程兄弟。修复还没部署就先跑了大样本。

## 正确做法

- **先看拓扑再绑核**：`lscpu -p=CPU,CORE,SOCKET`；HT 机器上给批处理整个 socket（如 `taskset -c 1-71:2`），生产独占另一个 socket
- **整棵进程树用 cgroup**：`systemd-run --scope -p MemoryLimit=100G -p CPUShares=256 <cmd>`（CentOS 7 / systemd 219 可用）；`ulimit -v` 只能做兜底
- **每个内存大户显式上限**：统一的 `duckdb_connect()` 助手设 `memory_limit`（低于 cgroup 上限，让它落盘到 `temp_directory`）、`threads`；fork 池进程数按内存算，不按核数算
- **先小样本看峰值再放大**：新代码路径（如新加的并行段）先用 0.5% 样本跑一次，记录 RSS 峰值，再上 6%
- 评审已指出的资源风险要**先修再跑**，不要"试跑先跑着，修复排队"
