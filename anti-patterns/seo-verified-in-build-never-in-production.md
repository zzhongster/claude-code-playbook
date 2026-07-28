# 反模式：SEO/GEO 基建只验本地产物，不验线上响应

## 一句话结论

预渲染、结构化数据、meta 这类"给机器看"的东西，在本地 `build/` 里验证通过**不等于**线上生效——它们不影响任何肉眼可见的功能，一旦漏部署或被覆盖，可以静默失效几个月无人察觉。验收动作只有一个：`curl` 线上 URL。

## 场景

任何做了 SSR / 预渲染 / SSG 的 SPA 项目。做完 `yarn build`，打开 `build/index.html` 一看内容齐全，任务标记完成。此后网站在浏览器里一切正常，没有任何报错、没有任何用户投诉。

## 发生了什么

某官网 2026-03 完成 react-snap 预渲染改造，本地产物验证无误。**该产物从未部署到生产**。

四个多月里没有任何人发现，因为：浏览器访问完全正常（SPA 的 JS 照常渲染），团队每次讨论"官网"看的都是浏览器。

2026-07 因为"国内 AI 被问及公司价格时经常乱写"来排查，第一次用 `curl` 拉线上首页：

```
$ curl -sL https://www.trade-ai.com/ | wc -c
649
```

**649 字节。** `<title>` 是 `React App`，`<meta name="description">` 是 `Web site created using create-react-app`，`lang="en"`。爬虫四个月来看到的一直是这个。

部署后同一条命令返回 248,777 字节。

## 数据支撑

上线当日验证 Search Console，回填的三个月历史数据正好构成"空壳期"对照组：

| 页面 | 展示 | 点击 | CTR | **平均排名** |
|------|------|------|-----|-------------|
| `/` | 2,585 | 1,294 | 50.06% | 2.94 |
| `/product` | 1,659 | 10 | **0.60%** | **2.00** |
| `/about` | 1,652 | 5 | **0.30%** | **1.84** |
| `/register` | 1,339 | 1 | **0.07%** | **1.27** |
| `/service` | 1,080 | 0 | **0%** | **1.37** |

**内容页合计 5,990 次展示、平均排名 1.27–2.00、只换来 16 次点击。**

搜索引擎早已把这些页排到首屏前两位——排名从来不是问题。问题是结果列表里那行标题写着 `React App`、描述写着 `Web site created using create-react-app`，没人会点。首页 CTR 能有 50%，纯粹因为搜品牌词的人认域名。

**流量三个月一直站在门口，只是门上没写字。** 而这件事在浏览器里看不出任何异常。

## 根因

1. **验证介质与消费者不匹配**。人用浏览器验收，而这批产物的唯一消费者是不执行 JS 的爬虫。用浏览器验证预渲染，等于用会读心术的观众验证字幕有没有打上去。
2. **静默失效**。缺失的 title/canonical/JSON-LD 不会报错、不会白屏、不会有用户反馈。反馈回路只存在于搜索引擎和 AI 的索引里，延迟以周计。
3. **"构建成功"被当成"上线成功"**。产物验证与部署验证是两件事，前者通过极易让人跳过后者。
4. **没人定期看爬虫视角**。团队每天都在看官网，但没有一次是用 `curl` 看的。

## 检测方法

**唯一可靠的动作：对线上 URL 发无 JS 的请求。**

```bash
# 1. 字节数——SPA 空壳通常只有几百字节，预渲染后应达数万
curl -sL -o /dev/null -w "%{size_download}\n" https://example.com/

# 2. 关键内容是否进初始 HTML（价格、标题等）
curl -sL https://example.com/pricing | grep -c "59,800"

# 3. meta 是否为业务文案而非框架默认值
curl -sL https://example.com/ | grep -o '<meta[^>]*description[^>]*>'

# 4. 以真实爬虫 UA 再验一次（CDN/WAF 可能对爬虫另眼相看）
curl -sL -A "Mozilla/5.0 (compatible; Bytespider; ...)" https://example.com/ | wc -c
```

注意 `-L`：预渲染产物是 `/pricing/index.html`，服务器通常会把 `/pricing` 301 到 `/pricing/`，不跟随重定向会误判失败。

## 正确做法

1. **把 curl 断言写进部署脚本**，让它成为发布流程的一部分而非人的自觉：

   ```bash
   # 发布后自动验证，字节数是最省事的哨兵
   size=$(curl -sL -o /dev/null -w "%{size_download}" "$LIVE_URL/pricing")
   [[ "$size" -gt 20000 ]] || echo "⚠️ 疑似退回 SPA 空壳，检查预渲染是否生效"
   ```

2. **构建阶段就拦截**。产物不全时直接 die，不要让半成品有机会上传（预渲染工具偶发崩溃很常见，如 react-snap 的 `Target closed`）：

   ```bash
   for page in index home pricing product service; do
     f=$([ "$page" = index ] && echo build/index.html || echo "build/$page/index.html")
     [[ -f "$f" ]] || die "预渲染产物缺失: $f"
   done
   grep -q "59,800" build/pricing/index.html || die "价格未进预渲染 HTML，SEO/GEO 会失效"
   ```

3. **接入站长平台建立外部反馈回路**。Search Console 之类的工具会持续告诉你机器眼中的站点是什么样，不必依赖人的定期自查。

4. **验收标准写成机器可判定的形式**。"预渲染做完了"不可验收；"线上 `/pricing` 的 HTML 含字符串 `59,800`"可验收。

## 适用范围

- **适用于**：SSR / 预渲染 / SSG 项目；结构化数据（JSON-LD）；canonical、hreflang、robots、sitemap；任何"给机器看、人看不见"的产物——也包括埋点、feed、API 契约文件
- **不适用于**：纯 CSR 且明确放弃搜索流量的内部系统

## 相关

- [预渲染序列化重排属性](prerender-serialization-reorders-attributes.md) — 同一项目的另一处静默失效
- [SPA 模板里硬编码页面级标签](spa-template-hardcoded-page-level-tags.md) — 也属于"不影响可见功能"的隐蔽 bug
