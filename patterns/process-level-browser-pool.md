# 模式：进程级浏览器池——整个进程一个 chromium，业务只借 context，出块必关

## 一句话结论

在长驻服务进程里用 Playwright，正确的形状是：**driver 和浏览器整个进程只起一份，业务代码通过 `async with page_context() as context` 借一个 BrowserContext**，信号量限制同时借出的数量，finally 里带超时关闭 context。这样没有任何一条业务异常路径能留下浏览器，并发也有上限；浏览器崩了下次借用自动重启。

## 场景

- FastAPI / aiohttp 等 asyncio 服务进程里跑后台抓图、渲染、截图
- 业务单元很多（每家公司、每个 URL 一次），每次都 launch 一个浏览器成本高（~1s）且泄漏风险大
- 容器环境，浏览器泄漏没人看得见

## 详细说明

### 结构（SQLAlchemy 风格的"池"，但只有一个浏览器）

```python
# app/workers/browser_pool.py
MAX_CONCURRENT_CONTEXTS = 2
CLOSE_TIMEOUT = 10.0

_sem = asyncio.Semaphore(MAX_CONCURRENT_CONTEXTS)
_lock: asyncio.Lock | None = None      # 懒建，避免绑到错误的 event loop
_pw = None                              # async_playwright().start() 的返回
_browser = None

async def _launch():
    """单独拆出来：测试时 monkeypatch 成假对象，不需要装 playwright"""
    from playwright.async_api import async_playwright
    pw = await async_playwright().start()
    try:
        browser = await pw.chromium.launch(args=[...], proxy=_proxy_config())
    except Exception:
        await _stop_quietly(pw)
        raise
    return pw, browser

async def get_browser():
    global _pw, _browser
    async with _get_lock():
        if _browser is not None:
            if _browser.is_connected():
                return _browser
            await _stop_quietly(_browser, "close")   # 断连/崩溃：收掉旧的再起
            await _stop_quietly(_pw)
        _pw, _browser = await _launch()
        return _browser

@asynccontextmanager
async def page_context(**context_kwargs):
    async with _sem:                                 # 并发上限
        browser = await get_browser()
        context = await browser.new_context(**context_kwargs)
        try:
            yield context
        finally:
            await _stop_quietly(context, "close")    # 带超时，异常吞掉：清理路径不能再抛

async def _stop_quietly(obj, method="stop"):
    if obj is None:
        return
    try:
        await asyncio.wait_for(getattr(obj, method)(), CLOSE_TIMEOUT)
    except Exception as e:
        logger.warning("%s.%s 失败/超时: %s", type(obj).__name__, method, e)
```

业务侧：

```python
async with page_context(viewport=..., user_agent=..., locale="en-US") as context:
    page = await context.new_page()
    await page.goto(url, timeout=15000)
    imgs = await page.evaluate(JS)      # 这里抛异常也没关系，context 在 finally 里关
```

### 配套三件

1. **启动清孤儿**：lifespan 里扫 `/proc/*/cmdline`，含 `chrome-headless-shell` / `ms-playwright` 的一律 SIGKILL（只在容器内做，`/.dockerenv` 存在才跑；开发机上别动别人的浏览器）。容器里没有 `ps`/`pkill`，`/proc` 是唯一可靠来源。
2. **关闭收浏览器**：lifespan 的 yield 之后 `await browser_pool.shutdown()`。
3. **健康接口暴露状态**：`/api/version` 带 `browser_alive` / `contexts_in_use` / `chrome_procs`，运维看 `chrome_procs` 不归零就是泄漏。

### 几个细节

- `_launch()` 单独拆一层，单测用假 browser/context 对象跑完整状态机，不需要在 CI 装 playwright
- `asyncio.Semaphore` / `asyncio.Lock` 在 Python 3.10+ 会绑定首次等待时的 loop；pytest 每个用例一个 loop，fixture 里重建它们
- `is_connected()` 是方法不是属性；断连时旧对象也要显式 `close()` 一次再丢
- 共享浏览器意味着代理是进程级的（launch 时定）；需要每个 context 不同代理时用 `launch(proxy={"server": "per-context"})` + `new_context(proxy=...)`
- `stats()` 里 `chrome_procs` 扫的是整个容器，测试脚本在容器里另起进程跑抓图时它会一起计入——这正是想要的

### 验证方法

单测（假对象）：正常/异常出块都关；6 个并发借用峰值 = 上限；断连后第二次借用 launch 计数 +1 且旧浏览器被 close；`close()` 卡死不外抛。

真机（容器内起独立进程跑一次完整抓图链路）：借用中 `chrome_procs` = 一个浏览器的子进程数（5），`shutdown()` 后 0。

## 数据支撑

trade-ai #437：改造前一次 25 家的补全留下 305 个 chrome 进程、整站 66 分钟不可用；改造后同一链路真机跑完 chromium 归零，发布后 1 小时 `chrome_procs` 持续为 0。单测 13 个、`tests/unit` 159 通过。

## 适用范围

- 适用于：asyncio 长驻进程里的 Playwright（同样思路适用于 Puppeteer / Selenium Grid 连接）
- 不适用于：一次性脚本；需要每次不同浏览器指纹/持久化 profile 的场景（那要按 profile 分池）
- 不能替代：把浏览器渲染挪到独立 worker 进程——池只是把爆炸半径限制在"抓图排队变慢"，不是"web 进程永远安全"

## 相关

- [反模式：web 进程里每次调用起一个浏览器](../anti-patterns/per-call-browser-launch-in-web-process.md)
- [反模式：无唯一约束的列上用 scalar_one_or_none](../anti-patterns/scalar-one-or-none-on-non-unique-columns.md)（同一周的另一起"后台任务整批失败"）
- trade-ai Issue #437 / PR #438
