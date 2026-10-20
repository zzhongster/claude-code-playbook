# 反模式：懒渲染组件吃掉预渲染正文——折叠面板里的答案一个字都没进 HTML

## 一句话结论

组件库的折叠面板 / 标签页默认**懒渲染**：未展开、未激活的那部分 children 根本不挂载。预渲染工具序列化的是首帧 DOM，于是产物里**只有标题没有正文**。而这件事的全部可见信号都是正向的——字节数涨了、可见文本字数涨了、页面数齐了、结构化数据条数对得上——**"完整正文已进 HTML"是假过的**。内容页必须逐条 `forceRender`；若目标是 AI 抓取，还要考虑 `innerText` 丢弃 `display:none` 的问题。

## 场景

- SPA 预渲染（react-snap / prerender-spa-plugin / puppeteer 自建）把首帧 DOM 存成静态 HTML
- 页面主体是折叠面板、手风琴、标签页、轮播、"展开更多"、虚拟列表——FAQ、文档、产品参数对比、规格表最典型
- 验收口径是"字数涨了""结构化数据 N 条齐了""页面都产出了"

## 详细说明

两层问题，严重性不同，必须分开看。

### 第一层：正文不在 HTML 里（硬缺陷）

antd `Collapse` 默认只渲染展开项的内容，`Tabs` 默认只渲染激活页。预渲染取的是首帧，所以未展开项的 children **从未存在过**，不是"隐藏了"。

```jsx
// 错：产物里 14 个问题、0 条答案
<Collapse items={items} defaultActiveKey={keys.slice(0, 3)} />

// 对：逐条声明 forceRender，children 一律挂载
<Collapse items={items.map(it => ({ ...it, forceRender: true }))} defaultActiveKey={…} />
// Tabs 对应的是 <Tabs items={[{ …, forceRender: true }]} />
```

**为什么字数不能当验收口径**：这一页改造前只渲染一行「暂无常见问题」（580 字），改造后 2,106 字。不加 `forceRender` 的中间态字数同样会涨——14 个问题标题就够让任何"字数比改造前多"的断言变绿。要断言的是**每一条正文都能在产物里命中**。

### 第二层：`display:none` 与 `innerText`（这是权衡，不是铁律）

`forceRender` 把正文放进了 DOM，但它仍在折叠起来的（`display:none`）容器里。两类消费者的行为不同：

| 消费者 | 读什么 | 看得到折叠内容吗 |
|---|---|---|
| 搜索引擎爬虫 | 解析 HTML / DOM | **看得到**（折叠内容照常索引） |
| 部分 AI 抓取管道 | 渲染后 `document.body.innerText` | **看不到**——`innerText` 按定义返回"渲染后可见文本"，丢弃 `display:none` 子树（这正是它与 `textContent` 的分水岭） |

如果一个页面存在的全部意义就是让后者读到答案，那就得让答案**默认可见**（本例把 `defaultActiveKey` 给了全量 14 条）。

**代价要摊开写，不要当成唯一正解**：页面变长、问题列表不便速览，而这是一次"工程替产品改了 UX"。备选是全展开 + 顶部锚点目录，或仍默认展开前几条（接受 `innerText` 管道读不到后面的）。不管选哪个，留一行可回退的代码并让人知道：

```jsx
// 默认全展开：部分 AI 抓取管道取 innerText，折叠内容对它们等于不存在。
// 想回到「默认展开前 3 条」：改这一行。
const defaultActiveKeys = collapseItems.map(i => i.key);
```

### 还有一条顺带的约束

结构化数据（FAQPage 等）里有、页面可见正文里没有，正是 Google 对 FAQPage 做**手动处罚**的靶子。所以"标记里的每一条都出现在可见正文中"不只是为了抓取，也是合规项——它刚好与第二层的断言是同一条。

## 怎么防

**三数一致 + 逐条命中**，别数字数：

```bash
# 1) 数据源条数 == 结构化数据 mainEntity 条数 == DOM 面板容器数
python3 - <<'PY'
import json, re, pathlib
html = pathlib.Path('build/faq/index.html').read_text()
# 先剥掉 <script>/<style>，否则 grep 到的是 CSS 文本而不是 DOM（见「相关」第一条）
dom = re.sub(r'<(script|style)\b[^>]*>.*?</\1>', '', html, flags=re.S|re.I)
qa  = json.load(open('src/data/faq.json'))
body = len(re.findall(r'ant-collapse-body', dom))
ld   = [json.loads(m) for m in re.findall(r'<script type="application/ld\+json">(.*?)</script>', html, re.S)]
faq  = next(b for b in ld if 'FAQPage' in json.dumps(b))
print(len(qa), len(faq['mainEntity']), body)          # 三个数必须相等
# 2) 逐条正文命中：比对前把两边全部空白去掉（<strong> 会把句子切段）
flat = re.sub(r'\s+', '', re.sub(r'<[^>]+>', ' ', dom))
miss = [q['q'] for q in qa if re.sub(r'\s+','',q['a']) not in flat]
assert not miss, miss
PY
```

把这段放进构建脚本或发布前闸门，而不是靠人每次去数。另外：**懒渲染的页面不要用"产物字节数"做哨兵**——它对这个缺陷完全不敏感。

## 数据支撑

| 口径 | 改造前（占位文字） | 不加 `forceRender` | 完成态 |
|---|---|---|---|
| 可见文本（本地产物口径） | 580 字 | 问题进、答案不进 | 2,106 字 |
| 产物里 `ant-collapse-body`（答案容器） | 0 | 0 | 14 |
| FAQPage `mainEntity` | 无 | 14（**标记有、正文无**） | 14 |
| 问答正文未进可见文本的条目 | — | 全部答案 | **0** |

生产站以 Baiduspider UA 直抓复核：可见文本 693 → 2,298 字，FAQPage 14 条且 **14/14 的问题与答案都出现在可见正文中**。

注意第三列：**结构化数据那一项是绿的，而页面上一条答案都没有**——这正是"标记合规检查通过 ≠ 内容真的在"的样子。

## 适用范围

- **适用于**：任何预渲染/SSG 下使用懒渲染容器的内容页（antd Collapse/Tabs、MUI Accordion、Element Plus、Bootstrap collapse、自写"展开更多"）；也适用于虚拟滚动列表——首帧只挂载视口内若干行，产物里就只有那几行
- **第二层（`innerText`/`display:none`）只在目标包含 AI 抓取管道时才成立**；纯搜索引擎 SEO 不需要为此牺牲 UX
- **不适用于**：SSR 框架里显式声明过 `ssr: true` 的组件，或本就全量渲染的实现

## 相关

- [用源码的写法去校验预渲染产物](prerender-output-verification-false-negatives.md) —— 上面那段校验脚本为什么要先剥 `<style>`、为什么要去全部空白
- [SEO/GEO 基建只验本地产物，不验线上响应](seo-verified-in-build-never-in-production.md) —— 本地产物过了还得 `curl` 线上
- [桌面/移动双轨样式靠特异性分胜负](dual-track-responsive-specificity-mismatch.md) —— 同一页同一轮改造的另一条
- [整份样式表失效，却因为页面上没东西而藏了半年](dead-stylesheet-hidden-by-thin-page.md) —— 答案从来没显示过，所以答案区的死样式也一直没人发现：两个缺陷互相掩护
