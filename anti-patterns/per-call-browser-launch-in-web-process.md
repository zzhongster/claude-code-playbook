# 反模式：在 web 进程里"每次调用起一个无头浏览器"，异常路径漏关，孤儿 chromium 把整站拖死

## 一句话结论

`async_playwright()` + `chromium.launch()` 写在每次抓图/渲染的函数里，看起来有 `finally: browser.close()` 也不安全：只要有一条异常路径绕过 close，或者 driver 停掉时 chromium 还没退干净，每次调用就留下一个浏览器。后台任务跑几十次，容器里几百个 chrome 进程，HTTP 进程的事件循环被喂死，连不碰数据库的健康检查都超时——外面看到的是"系统打不开"，而数据库、连接池、日志里的后台工作全都"正常"。

## 场景

- 后台任务（BackgroundTasks / asyncio.create_task）跑在 HTTP 进程里，任务里用 Playwright / Puppeteer / Selenium 渲染页面
- 每个业务单元（每家公司、每个 URL）单独 launch 一个浏览器，"用完就关"
- 异常处理写成 `try: ... except: return ""`，close 只在某几条路径上
- 容器精简镜像里没有 `ps` / `pkill`，泄漏了也没人看得见

## 详细说明

### 踩坑经历（trade-ai #437，2026-09-09）

买家推荐的「补全信息」对任务里 25 家公司逐一抓产品图：Firecrawl 找到候选页 → Playwright 渲染取图；Firecrawl 失败再走 Playwright 整站兜底。两条路径各自 `async with async_playwright() as p: browser = await p.chromium.launch(...)`。

13:56 用户点了一次补全，约 36 次浏览器启动。15:00 时容器里 319 个进程 / 1244 线程，305 个是 chrome-headless，`docker stats` CPU 105%。uvicorn 服务进程的主线程 py-spy 采样卡在 `playwright/_impl/_transport.py:run`。容器内直连 `localhost:8000/api/version` 8 秒超时，nginx 两小时记了 237 个 499。用户报"系统打不开"，66 分钟后重启恢复。

漏点两处：
1. 兜底路径把 `page.evaluate(...)` 写在 `try` 块外面，`goto` 成功但 `evaluate` 抛异常（页面跳转、上下文销毁）就直接跳到外层 `except`，`browser.close()` 永远不执行。
2. 主路径确实有 `finally: await browser.close()`，但 `async with async_playwright()` 退出会停掉 driver，chromium_headless_shell 常常没跟着死，被过继给容器 pid 1，永远活着。

### 为什么排查会走弯路

- 数据库完全健康（20 连接、无锁、无慢查询）——这不是连接池风暴，别往 DB 上查
- app 日志还在正常滚后台工作——进程"活着"，不是崩了
- 日志里 `watchfiles: 1 change detected` 每 30 秒一条，很像热重载风暴——其实生产命令虽然带着 `--reload`，uvicorn 只对 `*.py` 重载，18 小时内服务进程只启动过 1 次；那只是 uploads 落图被文件监视器记了一笔
- `docker stats` 的 CPU 是容器内所有进程之和，几百个 chromium 也能把它顶到 100%+，别误判成 Python 死循环
- 容器里没有 `ps`：扫 `/proc/*/status` 的 Name/Threads/PPid，脚本用 base64 传进去（ssh 单引号里套 heredoc 会吞）；`pip install py-spy` 在容器里能装，`py-spy dump --pid <uvicorn 子进程>` 一眼定性

### 替代方案

- 整个进程只允许一个浏览器，业务代码只借 context——见 [进程级浏览器池](../patterns/process-level-browser-pool.md)
- 任何"每个业务单元起一个 X"的后台循环都要有并发上限和入口幂等，否则一次点击就是一次 DoS
- 健康接口暴露 chromium 进程数（扫 `/proc/*/cmdline` 认 `chrome-headless-shell` / `ms-playwright`），运维看到它不归零就是泄漏
- 更根本的是把浏览器渲染从 HTTP 进程挪到 worker 进程；做不到之前至少做到上面三条

## 数据支撑

| 指标 | 事故中 | 重启后 | 修复后真机验证 |
|---|---|---|---|
| 容器内进程 / 线程 | 319 / 1244 | 134 / — | — |
| chrome-headless 进程 | 305 | 0 | 借用中 5（一个浏览器），shutdown 后 0 |
| `/api/version` 延迟 | >8s 超时 | 12ms | 12ms |
| docker stats CPU | 105% | 0.2% | — |

## 适用范围

- 适用于：Playwright / Puppeteer / Selenium 在长驻服务进程（FastAPI、Django+Celery worker 之外的 in-process 后台任务）里的任何用法
- 不适用于：一次性脚本——跑完进程退出，泄漏随进程一起消失

## 相关

- [进程级浏览器池](../patterns/process-level-browser-pool.md)
- trade-ai Issue #437 / PR #438（`server/app/workers/browser_pool.py`）
