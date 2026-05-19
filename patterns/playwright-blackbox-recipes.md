# Playwright Python 黑盒验收：7 个高频坑套路

> 来源：trade-ai v1.0 发版前一日跑通 7 个端到端黑盒（refresh-trade / group-research / GroupOverviewPanel / MergeLogicPanel accordion / TradeRecordsTable 翻页+约束 modal）—— 2026-05-20

适用：在跑 SPA + FastAPI 后端的 SSE 流、modal 交互、数据表分页、多候选 resume 等场景做端到端验收时。

---

## 1. Clash / 系统代理穿透 → 启动绕过

Mac 本机开 Clash Verge，环境变量 `HTTP_PROXY=http://127.0.0.1:7897` 会被 Chromium 继承，导致 `localhost:5173` 也走代理拿到 `ERR_PROXY_CONNECTION_FAILED` 或 502。

**三件套缺一不可**：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
```

```python
browser = pw.chromium.launch(
    headless=True,
    proxy={"server": "direct://"},
    args=["--no-proxy-server", "--proxy-bypass-list=*"],
)
```

`curl` 同样：`curl --noproxy '*' http://localhost:...`。

---

## 2. SSE 流 raw 抓取 + 解析

后端推 `data: {json}\n\n` 格式的 SSE 进度事件，playwright 用 `page.on("response")` 抓 body 后按行解析。

```python
sse_streams = []  # list 不是 str — resume 会触发第二条 SSE
def on_response(resp):
    if resp.url.endswith("/api/.../some-stream"):
        body = resp.body().decode("utf-8", errors="replace")
        sse_streams.append(body)
page.on("response", on_response)

# 解析
events = []
for line in raw.splitlines():
    if line.startswith("data: "):
        events.append(json.loads(line[6:]))
```

注意：`paused → preselected_xxx` 这类 resume pipeline 会触发第二条独立 SSE 请求，必须用 list 累加。

---

## 3. UI 完成自动折叠 → step 节点消失

类似 `ResearchTimeline` 这种"完成后自动折叠成 1 行"的组件：内部 `done=true && collapseOnDone` 一旦 effect 跑就把所有 step row 卸载 → 脚本以 `[data-research-step]` 计数变 0，但其实流程跑完了。

**两个解法（择一）**：

**A. 完成后点折叠 button 重展开重抓**

```python
collapsed = page.locator("[data-research-timeline='collapsed']").first
if collapsed.count():
    collapsed.click()
    page.wait_for_timeout(500)
    # 重新 query DOM 抓 step
```

**B. 跑中每 2 秒快照累加 `seen_done_steps`**

```python
snapshots = []
seen_done_steps = set()
while time.time() - start < MAX_S:
    step_data = page.evaluate("""() => Array.from(
        document.querySelectorAll('[data-research-step]')
    ).map(n => ({status: n.getAttribute('data-research-step-status')}))""")
    if step_data: snapshots.append(step_data)
    for s in step_data:
        if s.status == "done": seen_done_steps.add(s.title)
    ...
```

A 更稳，B 更全（能看到中间态）。两个一起最稳。

---

## 4. modal overlay 拦截 click → 必须 scope

modal 用 `<div class="fixed inset-0 z-50">` overlay 盖在外层 page 之上。如果脚本写 `button:has-text('应用')`，playwright 可能匹配到 modal 外的同名按钮，再点时被 overlay 拦截 → 30s timeout：

```
<div class="fixed inset-0 z-50">…</div> intercepts pointer events
- retrying click action ... (反复 60 次)
```

**正确做法：scope 到 modal 容器内**

```python
apply_btn = page.locator("div.fixed.inset-0.z-50 button").filter(
    has_text=re.compile(r"应用约束|确认|保存")
).first
```

或先 dump modal 里所有 button 看实际文案：

```python
modal_btns = page.evaluate("""() => {
    const m = document.querySelector('div.fixed.inset-0.z-50');
    return m ? Array.from(m.querySelectorAll('button'))
                  .map(b => b.textContent.trim()) : [];
}""")
```

---

## 5. 外部 API 非确定 → SSE 检测 + 重试

依赖外部数据源（BVD / 海关 / LLM 等）的 pipeline，同一输入两次跑可能：
- ① 5 候选 candidates_needed（happy path）
- ② `error: 未找到 BVD 记录`（依赖方 5xx / 限流）

脚本检测 SSE 流是否含 `error` 事件，自动重试最多 3 次：

```python
def has_error_in_sse():
    if not sse_streams: return False
    for line in sse_streams[-1].splitlines():
        if line.startswith("data: "):
            try:
                if json.loads(line[6:]).get("type") == "error":
                    return True
            except: pass
    return False

for retry in range(3):
    t0 = time.time()
    while time.time() - t0 < 50:
        page.wait_for_timeout(2000)
        if has_candidates_in_sse(): break
        if has_error_in_sse(): break
    if has_candidates_in_sse(): break
    if has_error_in_sse() and retry < 2:
        trigger_btn.click()
        page.wait_for_timeout(3000)
```

---

## 6. Vite dev alias 缓存紊乱 → 清 deps 重启

`vite dev` 长跑（数小时～数日）偶发 `web/.vite/deps` 内部状态坏，所有 `@/` alias import 都报：

```
Failed to resolve import "@/api/report" from "src/routes/print.tsx". Does the file exist?
```

文件实际存在、`tsconfig.json` paths 配置没变、其他文件同样写法能跑——只这一两个文件失败。直接复活：

```bash
kill <vite_pid>
rm -rf web/.vite/deps web/node_modules/.vite/deps
cd web && nohup pnpm dev > /tmp/vite.log 2>&1 &
sleep 5  # 等首次依赖优化
curl --noproxy '*' http://localhost:5173/src/routes/print.tsx  # 应 200
```

---

## 7. 测试账号 session token 直接复用

绕过 form 登录耗时：直接从 DB 拿现有 `session_token`：

```bash
docker exec server-db-1 psql -U trade_ai -d trade_ai -c \
  "SELECT session_token FROM users WHERE username='test_user';"
```

```python
ctx.add_cookies([{
    "name": "yc_session",  # 实际 cookie 名见 _SESSION_COOKIE
    "value": TOKEN,
    "domain": "localhost", "path": "/",
}])
```

⚠️ 只用专用测试账号，不动管理员账号（密码改了真人就进不去）。

---

## 8. 元素文案陷阱：先 grep 组件源码

凭直觉猜按钮文案翻车率很高。例如同一 trade-ai 项目里：

| 看截图猜的 | 实际源码 |
|---|---|
| 「选择」 | 「选这个」 |
| 「应用」 | 「✓ 应用约束」 |
| 「完成」 | 「已完成调研 · N 步」 |

**写脚本前 30 秒 grep 一下源码**，比反复跑改省 10 分钟。

```bash
grep -rE "选择该|选这个|应用|确认" web/src/components/.../*.tsx
```

类似地，body 里"完成"二字可能在 milestone "成立"等无关地方出现，判定 timeline 完成应该用更精确的「已完成{collapseLabel} · {N} 步」整串。

---

## 模板

七个套路打包到一个最小可运行的 playwright 黑盒模板里：[blackbox_template.py](../templates/playwright-blackbox-template.py)（待补）。

---

## 关联

- 反面：[blackbox = 业务真跑通+落库，不是 UI 接通+toast 占位](../anti-patterns/blackbox-toast-not-business.md)（待补）
- 反面：[写了用例不真跑就 commit](../anti-patterns/test-written-not-run.md)（待补）
