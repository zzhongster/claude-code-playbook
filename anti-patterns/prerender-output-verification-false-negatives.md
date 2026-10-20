# 反模式：用源码的写法去校验预渲染产物——四个假阴性，差点去"修"已经正确的页面

## 一句话结论

预渲染产物不是你写的 HTML，而是**浏览器序列化出来的 DOM**；校验它的脚本如果照源码的写法去匹配，会连续撞上一堆假阴性：正则数到的类名其实在 `<style>` 里、按书写顺序写的属性正则全部落空（属性被按字母序重排、Helmet 标签还多一个 `data-rh`）、去标签时把标签换成空格会切断被 `<strong>` 分段的句子。再加上一个环境坑：`serve -s build` 的 `-s` 把子路径重写到首页，**你验的根本不是目标页**。前两条在同一次产物验收里各报过一次假警（其中一条一度得出"description 0 条、canonical 缺失"），后两条是同一轮验收里总结出的归一化与环境规则。

## 场景

- 预渲染 / SSG 产物的机器化验收：断言正文完整、meta 齐备、结构化数据条数、DOM 结构
- 写一段 grep / 正则 / Python 去数产物，而参照物是源码里的写法
- 本地起个静态服务，用浏览器或 headless 看产物

## 四个假象

| 假象 | 实际原因 | 怎么判别 | 正确做法 |
|---|---|---|---|
| 数 `ant-collapse-item` 得到的条数远超预期（或非零而 DOM 里其实没有） | 命中的是**注入到 `<style>` 里的 CSS 文本**，不是 DOM 节点。cssinjs / styled-components 会把类名成片写进产物的 `<style>` | 把 `<style>`/`<script>` 剥掉再数一次，数字变了就是它 | **统计 DOM 之前一律先剥 `<script>` 与 `<style>`** |
| `<meta name="description" content=…>` 这类正则**一条都匹配不到**，误判成"description 0 条、canonical 缺失" | puppeteer 序列化 DOM 时属性按**字母序**重排（`content` 跑到 `name` 前面），自闭合 ` />` 被规范成 `>`；react-helmet 注入的标签还多带 `data-rh="true"` | 直接 `grep -o '<meta[^>]*description[^>]*>'` 看产物里它长什么样 | 别用属性顺序匹配。用单属性锚点 `<meta[^>]*name="description"[^>]*>`，或干脆解析 DOM（`BeautifulSoup`/`node:html-parser`） |
| "这句正文没进产物"——但它其实在 | 去标签时把标签**替换成空格**：`正文里<strong>这样</strong>加粗` 变成 `正文里 这样 加粗`，而源码里的原句没有那些空格，片段比对落空 | 把两边的空白全去掉再比一次，命中了就是它 | **两个问题要用两套归一化**：问"可见文本有多少字"→ 标签换空格；问"这句话在不在"→ 去掉**全部**空白再比 |
| 本地起服务验子页，看到的却是首页内容；或断言"预渲染没生效" | `serve -s`（`--single`）把**所有**路径重写到 `/index.html`，而且重写发生在文件查找之前——`build/faq/index.html` 存在也照样被绕过 | 同一个 URL 用不带 `-s` 的 serve 再取一次，内容变了就是它 | 验预渲染产物用**不带 `-s`** 的 `serve build`，或 `python3 -m http.server -d build`（注意后者把 `/faq` 301 到 `/faq/`，`curl` 要带 `-L`） |

## 数据支撑

`serve -s` 这一条当场复现过。构造 `build/index.html`（内容 `HOMEPAGE-SHELL`）与 `build/faq/index.html`（内容 `FAQ-PRERENDERED-PAGE`），三种起法各取同一组路径（serve 14.2.6）：

| 请求 | `serve -s build` | `serve build` | `python3 -m http.server -d build` |
|---|---|---|---|
| `/faq` | `HOMEPAGE-SHELL` ❌ | `FAQ-PRERENDERED-PAGE` ✅ | 301 → `/faq/` |
| `/faq/` | `HOMEPAGE-SHELL` ❌ | `FAQ-PRERENDERED-PAGE` ✅ | `FAQ-PRERENDERED-PAGE` ✅ |
| `/` | `HOMEPAGE-SHELL` ✅ | `HOMEPAGE-SHELL` ✅ | `HOMEPAGE-SHELL` ✅ |

`-s` 下**连带尾斜杠的 `/faq/` 也拿不到目标页**——这就是"预渲染好像没生效"的假象来源。讽刺的是：`-s` 正是为了让纯 SPA 的子路径可访问而存在的，而预渲染的目的恰恰是让它不再必要。

前两条的代价是实际发生过的：同一次产物验收里各报过一次假警，其中属性重排那条一度得出"description 0 条、canonical 缺失"的结论——**如果照这个结论去"修"，会往本来就正确的页面里再塞一份重复的 meta**。第三条（`<strong>` 切句）是那一轮逐条比对 88 个文本片段时定下的归一化规则，第四条是起静态服务时的环境约束；它们没有各自的假警记录，但错了同样会得出"正文缺失""预渲染没生效"的反向结论。

## 可复用的校验骨架

```python
import re, pathlib

raw = pathlib.Path('build/faq/index.html').read_text()

# 1) 统计 DOM 之前先剥掉 script/style
dom = re.sub(r'<(script|style)\b[^>]*>.*?</\1>', '', raw, flags=re.S | re.I)

# 2) 两套归一化，各回答一个问题
visible = re.sub(r'\s+', ' ', re.sub(r'<[^>]+>', ' ', dom)).strip()  # 字数/可见文本
flat    = re.sub(r'\s+', '', re.sub(r'<[^>]+>', ' ', dom))           # 片段是否存在

print(len(visible.replace(' ', '')))                  # 可见文本字数
assert re.sub(r'\s+', '', expected_sentence) in flat  # 这句话在不在

# 3) meta 用单属性锚点，不要假设属性顺序；结构化数据从 raw 里取（它在 <script> 里）
descs = re.findall(r'<meta[^>]*\bname="description"[^>]*>', dom)
lds   = re.findall(r'<script[^>]*application/ld\+json[^>]*>(.*?)</script>', raw, re.S)
assert len(descs) == 1, descs
```

注意第三条的细节：**结构化数据必须从未剥离的 `raw` 里取**（它本身就在 `<script>` 里），而 DOM 统计必须用剥过的 `dom`。同一个文件要同时持有两个版本。

## 适用范围

- **适用于**：任何 headless 浏览器序列化出的 HTML（react-snap、prerender-spa-plugin、puppeteer 自建）；以及任何"源码写法 ≠ 产物形态"的构建链
- **第四条（`serve -s`）适用于一切静态产物验收**，包括本来就有多页 HTML 的 SSG
- **不适用于**：直出字符串模板的框架（EJS、Handlebars、Next.js 静态 meta），它们不经过 DOM 往返，属性顺序与书写一致

## 相关

- [忽略预渲染的 DOM 序列化会重排属性](prerender-serialization-reorders-attributes.md) —— 同一个序列化事实的**对外**后果：站长平台按固定顺序做正则，验证会失败；本条是它的**对内**后果：你自己的校验脚本会假阴性
- [拿无头截图当手机验收结论](render-verification-false-alarms.md) —— 同类问题的另一半：渲染/截图工具自己的怪癖也会报假警
- [懒渲染组件吃掉预渲染正文](lazy-rendered-panels-missing-from-prerender.md) —— 这些归一化规则正是那条的验收脚本要用的
- [SEO/GEO 基建只验本地产物，不验线上响应](seo-verified-in-build-never-in-production.md) —— 产物验对了，还得对线上 URL 再验一遍
