# 反模式：改完配置立刻验证，结果打在旧的边缘状态上——于是去改一个本来就对的配置

## 一句话结论

在有 CDN / 边缘分发的链路上，重新部署后的第一轮验证会**新旧混杂**：同一批路径里，有的已按新配置响应，有的还是旧的。混杂结果比全错更迷惑——全错你会怀疑部署没生效，部分错你会怀疑配置逻辑本身，然后去改一个本来就对的配置。判别只需一步：URL 挂个随机 query 再打一次。

## 踩坑经过

2026-08-14，发布一份带口令的静态资料站（Cloudflare Workers 静态资源）。第一次部署后发现所有 `.html` 被 307 跳到无扩展名路径——这是 Cloudflare 静态资源 `html_handling` 的默认值 `auto-trailing-slash` 干的，判断正确，改成 `none` 重新部署。第二轮验证：

| 路径 | 结果 |
|---|---|
| `/html/index.html` | 200 |
| `/html/real-ui-audit/index.html` | 200 |
| `/html/real-walkthrough/main-nav-scout.html` | 200 |
| `/html/modules/global-trade.html` | **307** |
| `/html/modules/ai-engine.html` | **307** |
| `/html/inventory/index.html` | **307** |
| `/html/interactions/index.html` | **307** |

同样是 `index.html`，`real-ui-audit` 那个 200、`inventory` 那个 307；同样是模块页，有的通有的跳。看起来像是新配置只对一部分路径生效——差一点就去翻 `html_handling` 还有没有别的取值，或者怀疑要逐目录配。

实际上没再改任何东西：挂 `?cb=$RANDOM` 复打，四条全部 200；不挂 cache buster 再打一遍，也已经变成 200，响应头 `cf-cache-status: MISS`（旧条目已不在）。**那四条 307 是第一轮验证自己在边缘留下的旧状态**——具体是响应被缓存住了，还是新配置在各 PoP 之间传播有先后，从外部分不清，但对做法没影响：两种机制的应对是同一个。

## 为什么容易上当

- **命中的是你自己刚才那轮验证留下的东西**，不是别人的流量。所以"我刚部署完，这链接还没给过任何人，哪来的缓存"这个直觉是错的——制造缓存的正是验证本身。
- **curl 自己不带缓存**，很容易把 curl 的结果当成源站行为。
- 只看状态码看不出来。`cf-cache-status`、`age`、`x-served-by` 这些头才是判据，而习惯上验证只 `-w "%{http_code}"`。

## 做法

- 部署后**第一轮**验证一律挂 cache buster（`?cb=$RANDOM`），确认新行为；确认之后再用干净 URL 打一遍，验证真实用户路径。
- 结果不一致时，先看响应头判断是不是边缘状态，**再去改配置**。顺序反了就会改对的东西。
- 给用户的正式链接不带 cache buster。

## 适用范围

- 适用于：Cloudflare Workers / CDN 回源 / 任何带边缘缓存的静态分发；改完部署配置后的验收
- 不适用于：直连源站、无中间缓存的链路

## 相关

- [口令保护的静态站放边缘 Worker](../patterns/password-gated-static-site-on-edge-worker.md)
- [配置层确认 ≠ 端到端验证](../patterns/split-config-check-from-e2e-verification.md)
