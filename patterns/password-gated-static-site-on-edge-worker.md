# 模式：要口令才能看的静态资料，把内容搬进边缘 Worker，别放公读对象存储

## 一句话结论

把一份私密静态资料发成链接，有三条路：①公读对象存储 + 页面里加个密码框 ②对象设为私有 + 反代层持云厂商密钥签名回源 ③**内容作为边缘 Worker 的静态资源随 Worker 一起部署，口令在边缘校验**。①是密码剧场——内容已经躺在公网可直读的对象上，绕过前端直接取对象即可；②真能挡住，但代价是把能读写整个 bucket 的 AK/SK 交给另一个平台；③零密钥外流、无旁路，且 Cloudflare Workers 静态资源的限额（2 万文件 / 单文件 25 MiB）对绝大多数资料包绰绰有余。判据不是"哪条更安全"，是**内容有没有一条不经过你那道门的读取路径**。

## 场景

- 交接包、竞品逆向资料、内部报价单要发给同事或客户，里面有真实联系人、手机号、后台截图
- 对方是非技术用户，要"点开就看"：不能让他装东西，也不能发压缩包
- 手上已经有一条"发布成公开预览链接"的现成通道——这正是要停一下的地方。**"发布"和"私密发布"不是同一件事的两个参数，是两条不同的链路**，顺手复用那条通道就会掉进 ①

## 三条路对比

| 路线 | 口令是否真的挡得住 | 密钥外流 | 额外基建 |
|---|---|---|---|
| ① 公读对象 + 前端密码框 | **挡不住** | 无 | 无 |
| ② 私有对象 + 反代签名回源 | 挡得住 | 云厂商 AK/SK 进边缘平台 | 要实现签名 |
| ③ **内容进边缘 Worker + 边缘验口令** | 挡得住 | 无 | 静态资源随 Worker 部署 |

①里那个密码框拦的是"愿意走正门的人"。对象存储的 URL 一旦可推导（同一 bucket、同一前缀），密码框连减速带都算不上。

## 做法（Cloudflare Workers）

```toml
# wrangler.toml
[assets]
directory = "./public"          # 内容放 ./public/h/<siteId>/
binding = "ASSETS"
run_worker_first = true         # 关键，见下
html_handling = "none"          # 关键，见下
```

Worker 侧四件事：

1. **路由分流**：`/h/<siteId>/*` 走口令门，其余前缀保持原有行为，别的一律 404。
2. **口令进 secret**（`wrangler secret put`），不落本地文件；cookie 值取 `sha256(版本号|siteId|口令)`，改口令即让所有已发出的 cookie 立刻失效，不需要额外的会话表。
3. **失败关门**：secret 解析不出来时返回"没有任何站点可访问"，不能因为读不到配置就放行。
4. **站点未登记返回 404 而不是 401**，不透露某个 siteId 是否存在。

Cookie 用 `HttpOnly; Secure; SameSite=Lax`，`Path` 限定到 `/h/<siteId>/`；放行后的响应头设 `Cache-Control: private, max-age=600`——浏览器可缓存，任何共享缓存（CDN、公司代理）不得留存。

## 四个坑

- **`text/*` 缺 charset，中文整篇乱码**。静态资源层按扩展名推断出的 Content-Type 不带 charset（`.md` → `text/markdown`、`.txt` → `text/plain`、`.csv` → `text/csv`），浏览器只能猜编码，UTF-8 按 GBK 解析就是一屏「鍇ㄦ湅」。**HTML 因为自带 `<meta charset>` 不受影响，所以只测 HTML 页面永远发现不了它**——而"HTML 阅读层 + Markdown 原文"正是资料包最常见的结构，原文那一半全是裸文本。在放行处统一给缺 charset 的 `text/*` 补上即可。这个坑在换一条链路（对象存储直传）时已经踩过一次，换到边缘 Worker 又复发：**只要中间有一层按扩展名猜 Content-Type，它就会再来一次**。
- **`run_worker_first = true` 不能少**。默认是 false，Cloudflare 会在 Worker 之前直接吐出静态资源，口令门形同虚设——而且首页看起来一切正常，因为你测的那一下正好走了 Worker。
- **`html_handling = "none"` 不能少**。默认的 `auto-trailing-slash` 会把 `/a/b.html` 307 跳到 `/a/b`。站内链接若全是显式 `.html`，等于每开一页多绕一跳。关掉之后目录形式的路径（以 `/` 结尾）不再自动找 index.html，得由 Worker 自己补。
- **共用 Worker 的两个整体覆盖语义**：`wrangler deploy` 会把 assets 目录整体同步，本地 `public/` 不在就等于把所有已发布的私密站点删了；`wrangler secret put` 也是整体覆盖，新增站点时必须把旧站点的条目一起写回那段 JSON。

## 验证清单

门开在路由层，**漏掉一条前缀就等于没有门**，所以不能只测首页：

```bash
BASE="https://<域名>/h/<siteId>"
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/index.html"                       # 401
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/<深层截图>.png"                    # 401
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/<原文>.md"                        # 401
curl -s -o /dev/null -w "%{http_code}\n" -H "Cookie: hg_<siteId>=deadbeef" "$BASE/index.html"  # 401
curl -s -o /dev/null -w "%{http_code}\n" "https://<域名>/h/<猜错的siteId>/"         # 404
curl -s -o /dev/null -c /tmp/ck -X POST -d 'p=<口令>&next=/h/<siteId>/index.html' "$BASE/__auth"  # 303
curl -s -b /tmp/ck -o /dev/null -w "%{http_code}\n" "$BASE/index.html"            # 200
```

抽查要覆盖**每一类内容**（HTML / 图片 / Markdown 原文 / csv / txt / json），不是每一类路径深度；除状态码外还要看 `%{content_type}` 里 `text/*` 有没有 charset。共用 Worker 还要回归原有通道。刚部署完时边缘可能仍在按旧配置响应，见 [验证打在旧的边缘状态上](../anti-patterns/verify-immediately-after-deploy-hits-stale-edge-state.md)。

## 实测数据

2026-08-14，易查云重构交接包：757 个文件 / 39 MB（42 个 HTML 页面 + 241 张截图 + 21 张 SVG + 382 个抓包与脚本 + 全部 Markdown 原文），最大单文件 0.8 MB。首次部署上传 706 个资源耗时 49 秒，之后只改脚本的重新部署 5 秒。国内访问走 Cloudflare 边缘，与原有 OSS 预览链路同域名共存、互不影响。

## 适用范围

- 适用于：几十 MB 以内、几百到几千个文件的静态资料包；全员共用同一个口令；内容基本不变或整包重发
- 不适用于：需要按人区分权限、要审计谁看过什么（那需要真正的登录系统）；单文件超 25 MiB；内容要频繁增量更新（每次都是全量重新部署）
- 另需注意：若走的是未备案域名，企业微信历史上出现过拦截，发群前先自己在企微里点一次

## 教训

发布前先扫一遍内容里的真实个人信息。这次扫出账号手机号出现 20 次、以及一批从平台上取到的真实商务联系人邮箱——口令挡住了外人，但拿到口令的人全看得见。**口令保护的是边界，不是内容本身**；边界之内该不该有这些东西，是另一个问题，得单独问一次。

## 相关

- [配置层确认 ≠ 端到端验证](split-config-check-from-e2e-verification.md)
- [验证打在旧的边缘状态上](../anti-patterns/verify-immediately-after-deploy-hits-stale-edge-state.md)
