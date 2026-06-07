# 浏览器内无副作用验证：monkeypatch 全局对象验离线 / 验下载

> 来源：凯越FDE 全景地图加持久化后端时，需要验「断网仍能记录」和「导出 .md 内容不变」，但既不想真断网、也不想真往磁盘下文件 —— 2026-06-08

## 一句话结论

要验证「网络失败分支」和「文件下载内容」这类带副作用的前端行为，不要真去断网 / 真去下文件——在页面里临时 monkeypatch `window.fetch` / `window.Blob` / `URL.createObjectURL`，跑完立刻还原，验证全程留在页面内、确定、可清理。

## 场景

用 Chrome MCP 或 Playwright 对前端做端到端验收时，遇到这两类「难驱动、难清理」的分支：

1. **错误 / 离线分支**：代码写了「网络失败时回退 localStorage、状态显示离线」，但自动化里真把网卡关掉既麻烦又会顺带把页面本身搞挂（页面就是 http 托管的）。
2. **文件下载内容**：导出 `.md` / `.csv` / 截图走的是 `Blob + a.click()` 下载，自动化拿不到下载内容，真点了还在磁盘留一堆测试文件要清理。

两者的共同点：要验的是「程序产出的数据 / 走的分支」，而不是「浏览器下载子系统」本身。那就把真副作用替换成可观测的捕获。

## 详细说明

### 1) 验离线 / 错误分支：临时让 fetch 失败

不动网络，直接让 `window.fetch` reject，触发错误分支，断言「数据仍落 localStorage」+「状态变离线」，然后还原。

```js
(async () => {
  const realFetch = window.fetch;
  window.fetch = () => Promise.reject(new Error('simulated offline'));

  // 触发一次真实编辑（dispatch 真事件，不是直接改内部状态）
  const ta = document.querySelector('.noteZone textarea');
  ta.value += ' [离线补记]';
  ta.dispatchEvent(new Event('input', {bubbles:true}));

  await new Promise(r => setTimeout(r, 1100)); // 等过防抖 + flush

  const result = {
    status: document.getElementById('saveState').textContent,           // 期望「仅本机（离线）」
    saved : JSON.parse(localStorage.getItem('k_notes')).x.includes('[离线补记]'), // 期望 true
  };
  window.fetch = realFetch; // 务必还原
  return result;
})();
```

**一气呵成验「断网→恢复」**：上面还原 fetch 后，再编辑一次 → 断言后端这次拿到了之前那条离线补记。一个流程同时覆盖「离线降级」和「恢复后补传」两个验收点。

### 2) 验下载内容：捕获 Blob，不落盘

导出按钮内部是 `new Blob(parts) → URL.createObjectURL → a.click()`。把 `Blob` 换成捕获 `parts` 的壳、把 `createObjectURL` 打桩，点按钮，拿到本该被下载的文本，眼检格式。磁盘上什么都不会多出来。

```js
const realBlob = window.Blob, realCreate = URL.createObjectURL, realRevoke = URL.revokeObjectURL;
let captured = null;
window.Blob = function(parts, opts){ captured = parts.join(''); return new realBlob(parts, opts); };
URL.createObjectURL = () => 'blob:stub';
URL.revokeObjectURL = () => {};

document.getElementById('btnExport').click();

window.Blob = realBlob; URL.createObjectURL = realCreate; URL.revokeObjectURL = realRevoke;
captured; // ← 这就是导出的 .md 全文，直接核对格式 / 是否含同步进来的数据
```

这一招特别适合验「重构后导出格式必须一字不变」这种硬约束：拿到全文直接和预期比。

### 3) 配合「真事件 + 双端断言」

monkeypatch 只替换副作用边界，**输入仍要走真实交互**（`dispatchEvent('input')` / `.click()`，而非直接写内部变量），输出要查**可观测状态**（DOM 文本、localStorage、还原后真 POST 到的后端）。这样验的才是用户路径，不是你脑补的路径。参见 [[playwright-blackbox-recipes]]、[[persona-driven-testing]]。

## 坑

- **一定要还原**：用 try/finally 或在 return 前还原被 patch 的全局，否则后续验证全跑在假 fetch 上，结果全错且难排查。
- **Chrome MCP 的 `javascript_tool` REPL 不支持顶层 await**：会报 `await is only valid in async functions`。要 await 就包一层 `(async () => { ... })()`。
- **别用它替代真集成测试**：跨标签页同步、后端真往返这类，还是要真 GET/POST 验（本项目就是 A 标签编辑→B 标签加载，两边都查真后端）。monkeypatch 只用于「真副作用难驱动 / 难清理」的那几个分支。
- **patch 范围越小越好**：只换 `fetch` 的某次调用语义、只捕获 `Blob`，不要顺手 stub 一大片，否则验证本身的可信度下降。

## 适用范围

- 适用于：前端「网络失败回退」「下载文件内容」「剪贴板写入」等带副作用、自动化难直接驱动的分支验证；Chrome MCP / Playwright 注入 JS 的场景。
- 不适用于：能直接用真 HTTP 往返、真跨标签页验证的集成点（那种就老实跑真的）；以及副作用本身就是被测对象时（比如要验浏览器真的触发了下载事件）。

## 相关

- [[playwright-blackbox-recipes]] — 黑盒验收的事件/SSE/代理穿透套路
- [[persona-driven-testing]] — 用真实用户路径而非 DOM 计数来验
- foundations/03-verification — 验证优先于宣称「完成」
