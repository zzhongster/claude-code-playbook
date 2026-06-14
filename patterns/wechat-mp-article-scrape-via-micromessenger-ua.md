# 微信公众号文章抓取：curl + 微信 UA 绕过"环境异常"验证

## 一句话结论

WebFetch / 普通 HTTP 抓 `mp.weixin.qq.com` 文章会触发「环境异常，完成验证后即可继续访问」反爬；换成 `curl` 带**微信 iPhone User-Agent** 即可直接拿到完整正文 HTML，无需任何 token 或登录。

## 场景

批量抓取微信公众号永久链接文章（`https://mp.weixin.qq.com/s/xxxx`）的正文，用于数据提取/归档时。

## 详细说明

- **失败**：WebFetch（及不带 UA 的 curl）被识别为非微信环境，返回的 HTML 只有一句「环境异常，完成验证后即可继续访问」，正文为空。
- **成功**：curl 伪装成微信内置浏览器的 UA，服务端直接返回完整页面。

```bash
UA="Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/8.0.20(0x18001432) NetType/WIFI Language/zh_CN"
curl -s -L -A "$UA" "https://mp.weixin.qq.com/s/xxxx" -o article.html
# 单篇约 3MB（完整页面）；返回几 KB 多半是被拦的验证页
```

- **正文结构**：正文在 `<div id="js_content">`，标题在 `<h1 class="rich_media_title">`。
- **图片是懒加载**：真实地址在 `<img data-src="...">`，`src` 往往是占位符——取 `data-src`。
- **提取**：`bs4` 即可（`html2text` 非必须）。`content.get_text("\n", strip=True)` 拿文字，`find_all("img")` 拿图片 URL。
- **完整流程**：`curl 下 HTML` → `bs4 取 js_content 文本 + 图片 URL` → 关键表格/图片**另行下载做视觉识别**（见 [[wechat-mp-data-in-images-not-text]]）。
- 下载图片时只下 `jpg/png`（`wx_fmt=png/jpeg`），跳过 `wx_fmt=gif`（多为装饰/分隔线），并丢弃 <3KB 的微型图标。
- **`data-src` 里的 URL 是 HTML 转义过的**，含 `&amp;`，下载前必须还原成 `&`（`sed 's/&amp;/\&/g'`），否则 `wx_fmt` / `from=appmsg` 等 query 参数错乱、可能取不到原图。
- **下载图片带 `-H "Referer: https://mp.weixin.qq.com/"`** 更稳妥：`mmbiz.qpic.cn` 有防盗链，带上来源更不易被拒。
- **正文图按 `data-src` 出现顺序 == 文档自然顺序**：多文档合辑（如"各科试卷"）里，矮图（高度远小于正文页）往往是章节分隔标题图，可据此把图片切成各章节段落。
- 标题不在 `<title>`（常为空），在 JS 变量 `var msg_title = '...'` 或 `<meta property="og:title">`。

```bash
# 提取正文图 → 解码 &amp; → 带 Referer 下载
grep -oE 'data-src="https://mmbiz.qpic.cn[^"]*"' article.html \
  | sed -E 's/data-src="//; s/"$//; s/&amp;/\&/g' > urls.txt
i=0; while read u; do i=$((i+1)); curl -s -A "$UA" \
  -H "Referer: https://mp.weixin.qq.com/" "$u" -o "img-$i.png"; done < urls.txt
```

## 适用范围

- **适用于**：公开的图文永久链接（`/s/` 短链）。
- **不适用于**：需登录/付费、链接本身带临时 token（易过期）的场景；大批量高频抓取可能被限流，必要时加请求间隔。

## 相关

- [[wechat-mp-data-in-images-not-text]] — 抓下来后，关键数据往往在图片里
- [[llm-extract-not-recall]]
