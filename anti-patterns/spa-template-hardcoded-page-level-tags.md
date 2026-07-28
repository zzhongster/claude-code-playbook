# 反模式：把页面级标签硬编码进 SPA 的 index.html

## 一句话结论

SPA 的 `index.html` 是**所有路由共用的模板**。写在里面的 canonical、og:url 这类页面级标签会被全站每一页继承——其中 canonical 的后果最严重：每个子页都在对搜索引擎宣称"我不是权威版本，首页才是"，等于主动放弃独立收录。这类 bug 潜伏期极长，因为它不影响任何肉眼可见的功能。

## 场景

CRA / Vite 等模板型 SPA。为了"补齐 SEO"，有人在 `public/index.html` 的 `<head>` 里加了一批 meta：

```html
<meta name="description" content="..." />
<link rel="canonical" href="https://www.example.com" />
<meta property="og:url" content="https://www.example.com" />
```

看起来很合理——站点级的默认值嘛。子页面再用 react-helmet 覆盖各自的 title 和 description。

## 发生了什么

某官网做完预渲染上线后复查，发现 9 个路由的 canonical **全部指向首页**：

```
/            → https://www.trade-ai.com
/pricing     → https://www.trade-ai.com   ← 刚做完的价格页
/product     → https://www.trade-ai.com
/privacy-en  → https://www.trade-ai.com
```

根因就是 `public/index.html:11` 那一行硬编码。各页面的 Helmet 只覆盖了 title 和 description，**没有人想到 canonical 也需要逐页声明**。

后果：`rel="canonical"` 是页面对搜索引擎的自我声明——"本页的权威版本是 X"。价格页声明自己的权威版本是首页，搜索引擎据此可以判定它是首页的重复内容，不予独立收录，或把它的权重并给首页。

**为了让 AI 能检索到价格而做的全部预渲染工作，被这一行抵消掉大半。**

`og:url` 是同一个毛病，只是后果轻些（影响社交平台分享卡片指向）。

## 为什么这类 bug 能长期存活

- 网站在浏览器里完全正常，没有报错、没有白屏
- 每个页面**确实有** canonical 标签，粗看"SEO 做了"，不打开看值不会发现
- 页面级 title/description 通常有人管（因为浏览器标签栏可见），canonical 无人可见、无人过问
- 各种 SEO 检查清单往往只问"有没有 canonical"，不问"指向对不对"

## 正确做法

**页面级标签必须运行时按路由生成**，模板里只保留真正站点级的东西。

删掉模板里的硬编码，并留下为什么不能加回来的注释：

```html
<!--
  canonical 与 og:url 不在此处硬编码：这份 index.html 是所有路由共用的模板，
  写死首页地址等于让每个子页面都声明"首页才是我的权威版本"，
  搜索引擎会据此把子页当重复内容、不予独立收录。
  改由 src/layouts/index.jsx 按当前路由动态输出。
-->
```

在布局组件里统一生成，一处覆盖全部子路由，比逐页添加更不容易漏：

```jsx
const SITE_ORIGIN = "https://www.example.com";

// 去掉尾斜杠：预渲染产物是 /pricing/index.html，浏览器路径可能是 /pricing 或 /pricing/，
// 两者须归一，否则首帧不一致会触发 hydration 失败
const normalizePath = (p) => (p.length > 1 ? p.replace(/\/+$/, "") : p);

// / 与 /home 渲染同一组件、内容完全一致，统一规范到根路径，消除这组重复内容
const getCanonicalUrl = (pathname) => {
  const n = normalizePath(pathname);
  return SITE_ORIGIN + (n === "/home" ? "/" : n);
};

// 组件内
const canonicalUrl = getCanonicalUrl(useLocation().pathname);
<Helmet>
  <link rel="canonical" href={canonicalUrl} />
  <meta property="og:url" content={canonicalUrl} />
</Helmet>
```

⚠️ **必须删掉模板里的静态标签**。react-helmet 只管理自己注入的标签，不会移除 HTML 里已有的同名标签——两者共存会产生**两条 canonical**，比一条错的更糟（搜索引擎可能两条都忽略）。

## 检测方法

一条命令查全站，重点看**条数**和**指向**：

```bash
for p in / /pricing /product /about /news; do
  printf "%-12s " "$p"
  html=$(curl -sL "https://example.com$p")
  echo "$(echo "$html" | grep -c 'rel=\"canonical\"') 条 | $(echo "$html" | grep -o '<link[^>]*canonical[^>]*>' | head -1)"
done
```

判据：**每页恰好 1 条，且指向自己**。出现 0 条、2 条、或全部指向同一个 URL，都是问题。

## 哪些标签属于"页面级"

| 层级 | 标签 | 放哪 |
|------|------|------|
| 页面级 | `canonical`、`og:url`、`title`、`description`、`og:title`、`og:description`、`hreflang`、JSON-LD | 运行时按路由生成 |
| 站点级 | `charset`、`viewport`、`theme-color`、favicon、`og:site_name`、站长平台验证码 | 模板里硬编码没问题 |

判断标准很简单：**这个值会不会因页面而异？** 会，就不能进模板。

## 适用范围

- **适用于**：所有基于单一 HTML 模板的 SPA（CRA、Vite、Vue CLI 等），无论是否做了预渲染
- **不适用于**：每个路由独立产出 HTML 的框架（Next.js、Nuxt、Astro 等），它们的页面级 API 天然按页隔离

## 相关

- [SEO 基建只验本地产物，不验线上响应](seo-verified-in-build-never-in-production.md) — 同类的静默失效，检测手段相同
- [预渲染序列化重排属性](prerender-serialization-reorders-attributes.md) — 另一处只有 curl 才看得见的问题
