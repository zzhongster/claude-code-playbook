# 反模式：自动化工具留下的孤儿 headless 浏览器空转烧 CPU，而「关掉浏览器」对它完全无效

## 一句话结论

Agent / 爬虫 / e2e 框架拉起的 headless Chrome 一旦变成孤儿进程（父进程退出、PPID=1），它**没有窗口、不在 Dock、不在任何界面里**，用户关掉自己那个 Chrome 一点用没有；配上软件渲染和一个已经死掉的本地代理端口，它能持续吃满 8 个核心跑几十小时无人察觉。

## 场景

- CPU 长期跑满、风扇狂转，但「我该关的都关了」
- 活动监视器里看到一堆 `Google Chrome Helper`，可用户确信自己没开 Chrome
- 用过 puppeteer / playwright / 各类 agent-browser / MCP 浏览器工具 / webapp-testing 类 skill
- 系统 `idle` 接近 0，且 `sys`（系统态）占比异常高

## 详细说明

### 2026-08-28 实测

某台 10 核 Mac（4 性能核 + 6 能效核）持续满载。用户已退出 Chrome，但进程仍在。

```
$ ps -Ao pid,ppid,pcpu,etime,command -r | head
85218 85210  99.9  01-07:12:24  ...Google Chrome Helper (Renderer)
85214 85210  99.3  01-07:12:24  ...Google Chrome Helper (GPU)
85249 85210  95.1  01-07:12:12  ...Google Chrome Helper
（同族 13 个，合计 823.7%）
```

顺着 PPID 往上刨：

```
85210 ← 85178 ← 1(launchd)
       └─ ~/.workbuddy/.../node_modules/agent-browser/bin/agent-browser-darwin-arm64
```

**PPID=1 是决定性信号**：拉起 agent-browser 的那个会话早就退了，它成了孤儿，没有任何人会来回收它。已运行 **31 小时 13 分**。

### 三个让它「隐身 + 空转」的启动参数

```
--headless=new                       # 无头：没有窗口，关 Chrome 碰不到它
--enable-unsafe-swiftshader          # GPU 走 CPU 软件模拟 → GPU helper 常驻 ~100%
--proxy-server=http://127.0.0.1:49660
--user-data-dir=/var/folders/.../agent-browser-chrome-<uuid>
```

第三行是最值得查的一行：

```
$ lsof -nP -iTCP:49660 -sTCP:LISTEN
（零输出——这个代理端口已经没有任何进程在监听）
```

**（推断，非实证）** 所有网络请求打进一个死端口后不断失败重试，是它空转的直接来源。这一条没有抓包或日志佐证，只有「端口无监听」这个事实；但即便去掉这条推断，前两行参数已足以解释常驻高占用。

第四行是好消息：profile 在临时目录里，是一次性的，**杀掉不丢任何用户数据**，和你自己那个 Chrome 的书签/登录态完全无关。

### SIGTERM 杀不动，且会短暂恶化

```
$ kill 85178 85210        # 部分子进程响应了，主进程纹丝不动
# 3 秒后再看：renderer 从 31% 飙到 70%
$ kill -9 <整棵树>        # 才清干净
```

进程已经卡到收不下正常退出信号。**别在 SIGTERM 之后干等**，看一眼没退就上 `-9`。

### 附带发现：Docker Desktop 的 VM 空转与容器负载无关

同一台机器上第二大户是 Docker 的虚拟机进程（`com.apple.Virtualization.VirtualMachine.xpc`），开了 46 天。

停掉 6 个容器里的 5 个之后：

```
停止前： PID 1393  CPU 102.0%  MEM 4268M
停止后： PID 1393  CPU 102.0%  MEM 4268M      ← 两个数一模一样
```

**这说明它烧的不是容器负载。** 硬证据用 CPU 时间增量法拿（见下节）。同时 `docker stats` / `docker inspect` / `docker version` 全部超时——daemon 已经半死。重启 Docker Desktop 后 daemon **6 秒**就绪，新 VM 掉到 0.1%。

⚠️ 重启 Docker 后**容器不一定自动回来**：restart policy 不是 `always` 的容器会停在 `Exited (0)`，要手动 `docker start <name>`。重启前如果 daemon 已经不响应，你**查不到** restart policy，所以默认假设要手动拉。

## 数据支撑

| 阶段 | idle | Load Avg(1min) | sys 占比 |
|---|---|---|---|
| 初始 | **0.0%** | 28.46 | 68.86% |
| 杀掉孤儿 headless Chrome（-823%） | 28.16% | 18.36 | 25.48% |
| 停 5 个闲置容器 | **74.96%** | 5.34 | 4.63% |
| 重启 Docker Desktop（VM 从 102% → 0.1%） | 稳态回落 | — | — |

**CPU 时间增量法**——判断一个进程是不是「真的在持续烧」，比任何 %CPU 读数都硬，且不受采样口径影响：

```bash
t1=$(ps -o time= -p <PID>); top -l 4 -s 5 -n 0 >/dev/null; t2=$(ps -o time= -p <PID>)
echo "$t1 -> $t2"
# 实测 Docker VM: 19365:10.83 -> 19365:26.88
# 16.05 秒 CPU 时间 / 约 15-20 秒墙钟 ≈ 持续占满一个核心
```

（用 `top -l 4 -s 5` 制造间隔而不是 `sleep`，是因为部分 agent 环境会拦截前台裸 `sleep`。）

## 适用范围

- 适用于：macOS / Linux 上任何由自动化工具拉起的 headless 浏览器；同理适用于孤儿 node、python worker、Docker VM
- 适用于：「关了 App 但进程还在」这一类现象
- 不适用于：有可见窗口的正常浏览器占用高（那是页面本身的问题，去查标签页）

## 排查顺序（30 秒）

```bash
# 1. 有没有看不见的 headless 浏览器
ps -Ao pid,ppid,pcpu,etime,command | grep -- "--headless" | grep -v grep

# 2. 是不是孤儿（PPID=1 = 没人管它了）
ps -o pid,ppid,etime,command -p <PPID>

# 3. 它的本地代理端口还活着吗
lsof -nP -iTCP:<port> -sTCP:LISTEN

# 4. 确认在持续烧而不是历史峰值 → CPU 时间增量法（见上）
```

## 相关

- [清理后立刻采样，把饥饿反弹误读成新元凶](post-cleanup-cpu-rebound-misread.md)——本次排查里紧接着踩的第二个坑
- [装带本地 daemon 的 Skill 不配套关闭手段](skill-with-local-daemon.md)——**同一个病的另一个后果**。那条讲「不写关闭脚本 = 留了个零认证本地后门」（安全面），这条讲同一批进程漏出来之后会**吃满 CPU 几十小时无人察觉**（资源面）。判据是同一条：**装一个会起后台进程的工具时，先问「用完怎么关、关不干净会怎样」**
