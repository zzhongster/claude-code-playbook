# 反模式：忽略预渲染的 DOM 序列化会重排属性

## 一句话结论

puppeteer 类工具（react-snap、prerender-spa-plugin 等）输出的是**序列化后的 DOM**，不是你写的 HTML——属性顺序会被重排。HTML 语义完全等价，但站长平台的所有权校验普遍是**按官方示例的固定顺序做正则匹配**，顺序一反就验证失败，而平台只会告诉你"验证失败"，不给任何原因。

## 场景

给站点接入头条搜索 / 百度 / Google 等站长平台，选 HTML 标签验证方式。把平台给的 meta 原样贴进 `public/index.html`，构建、部署、点验证——**失败**。

肉眼检查线上页面：标签在，值也对，一个字符没差。

## 发生了什么

某站点接入头条搜索站长平台，首次验证失败。逐字符比对后才发现：

```html
<!-- 源文件 public/index.html，与平台官方示例一致 -->
<meta name="bytedance-verification-code" content="hvT5/h5/KXbieN6hXqhd" />

<!-- 线上产物（react-snap 输出） -->
<meta content="hvT5/h5/KXbieN6hXqhd" name="bytedance-verification-code">
```

`content` 被排到了 `name` 前面——puppeteer 序列化 DOM 时属性按字母序输出，页面上**所有** meta 都被这样重排了（`description`、`og:*` 一个不落，只是它们由搜索引擎按 DOM 解析，顺序无关，所以从没暴露过问题）。

自闭合的 ` />` 也被规范成了 `>`。

平台的校验器多半是一条形如 `<meta\s+name="bytedance-verification-code"\s+content="([^"]+)"` 的正则。对正则来说，这就是两个不同的字符串。

## 根因

**"我写的 HTML"和"线上的 HTML"之间隔着一层序列化，而人往往假设它是恒等变换。**

浏览器把 HTML 解析成 DOM 时，属性只是一个无序集合——顺序信息在这一步就丢了。`outerHTML` 再序列化回字符串时，顺序由实现决定。整条链路完全符合规范，只是不保留你的书写形式。

而消费方分成两类，对此的容忍度天差地别：

| 消费方 | 解析方式 | 属性顺序 |
|--------|---------|---------|
| 浏览器、搜索引擎爬虫 | DOM 解析 | 无关 |
| **站长平台所有权校验** | **正则匹配** | **强相关** |

## 排查提示

站长平台验证失败时，按这个顺序查：

1. **逐字符比对线上产物与官方示例**，而不是"看看标签在不在"：

   ```bash
   curl -sL https://example.com/ | grep -o '<meta[^>]*verification[^>]*>'
   ```

   把结果和平台给的示例并排放，比对整个字符串——顺序、自闭合斜杠、空格。

2. **爬虫来源 IP 是否被 CDN / WAF / 高防拦截**。UA 可以自己模拟测，IP 段测不了，得去防护控制台确认放行。

3. **CDN 缓存**是否还在返回旧版页面。

4. 以上都不成立，**改用文件验证**——静态文件不经过前端框架和预渲染，绕开全部序列化问题，比 meta 标签可靠得多。

## 正确做法

在预渲染**之后**加一道后处理，只还原站点验证类标签的书写顺序：

```js
// package.json: "postbuild": "react-snap && node scripts/fix-verification-meta.js"

const VERIFY_META_NAMES = [
  "bytedance-verification-code",  // 头条搜索
  "baidu-site-verification",
  "google-site-verification",
  "sogou_site_verification",
  "360-site-verification",
  "msvalidate.01",                // Bing
];

for (const file of collectHtmlFiles(BUILD_DIR)) {
  let content = fs.readFileSync(file, "utf8");
  for (const name of VERIFY_META_NAMES) {
    // 匹配被重排成 content 在前的形态，还原为 name 在前
    const reordered = new RegExp(`<meta\\s+content="([^"]*)"\\s+name="${name}"\\s*/?>`, "g");
    content = content.replace(reordered, (_, v) => `<meta name="${name}" content="${v}" />`);
  }
  fs.writeFileSync(file, content, "utf8");
}
```

**只处理验证类标签，不要顺手把所有 meta 都改回去。** `description`、`og:*` 由搜索引擎按 DOM 解析，属性顺序对它们毫无影响；为个别场景重写全部产物，只会引入新的不确定性。

配一条构建期断言，把"和官方示例逐字符一致"变成机器可判定的条件：

```js
const want = '<meta name="bytedance-verification-code" content="hvT5/..." />';
const got = /<meta[^>]*bytedance[^>]*>/.exec(html)?.[0];
if (got !== want) throw new Error("验证标签形态与官方示例不一致");
```

## 适用范围

- **适用于**：任何用 headless 浏览器序列化 DOM 产出 HTML 的方案——react-snap、prerender-spa-plugin、puppeteer 自建预渲染、部分 SSG 的 hydration 快照
- **同类风险**：其他按正则匹配 HTML 的第三方校验——广告平台落地页审核、支付回调域名校验、某些安全扫描器
- **不适用于**：直出字符串模板的框架（EJS、Handlebars、Next.js 的静态 meta），它们不经过 DOM 往返

## 相关

- [SEO 基建只验本地产物，不验线上响应](seo-verified-in-build-never-in-production.md) — 同一项目、同一天发现的另一处静默失效
- [SPA 模板里硬编码页面级标签](spa-template-hardcoded-page-level-tags.md)
