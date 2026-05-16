# 反模式：mihomo geoip 库对国际大厂跨境 CDN/接入节点 IP 段归属判断不可靠

## 一句话结论

mihomo（Clash Meta 内核）内置的 geoip 数据库对**国际大厂故意放在跨境区的接入节点 IP** 经常错标，靠 `GEOIP,CN` + `MATCH` 兜底的分流策略会**双向出问题**——本该走代理的（被错标为 CN）走了直连、本该直连的（被错标为非 CN）走了代理。后果是被目标网站按 IP 风控（地理跳转、异地登录阻断、消息丢失）。

## 场景

典型的"够用就好"型分流配置：

```yaml
rules:
  # 显式列出几个常用国外网站走代理
  - DOMAIN-SUFFIX,google.com,🌍 国外网站
  - DOMAIN-SUFFIX,github.com,🌍 国外网站
  - DOMAIN-SUFFIX,youtube.com,🌍 国外网站
  # ...

  # 局域网直连
  - IP-CIDR,192.168.0.0/16,DIRECT

  # 国内 IP 直连
  - GEOIP,CN,🇨🇳 国内网站

  # 其他默认走代理
  - MATCH,🐟 漏网之鱼
```

逻辑看起来很合理：国外站显式走代理、国内 IP 走直连、其他兜底走代理。但只要域名没显式列出、且 IP 段被 geoip 库错标，分流就翻车。

## 踩坑经历

### Case A：LinkedIn 被错标为 CN（2026-05-13）

员工反馈：浏览器访问 `linkedin.com` 强制跳转到 `linkedin.cn`，无痕窗口也跳。

排查：

1. **VPS 直 curl**：从日本出口 IP curl `linkedin.com`，HTTP/2 200，**HTML 里搜不到任何 `linkedin.cn` 字符串**——服务端基于日本 IP 不会跳 .cn
2. **客户端连接面板**：`www.linkedin.com:443` 命中规则 **`GeoIP(cn)`** → 🇨🇳 国内网站 → **DIRECT**；同时 `content.linkedin.com` 和 `platform.linkedin.com` 走的是 `MATCH → 漏网之鱼 → JP-Reality` 正常代理

也就是说 mihomo geoip 库把 **`www.linkedin.com` 解析到的 IP 段标为 CN**（LinkedIn 历史上在中国跟世纪互联合作部署过 PoP，IP 注册信息没清干净），但同公司的 content / platform 子域的 IP 没被错标。流量从本地中国 IP 直连出去 → LinkedIn 服务端按 IP 跳 `.cn`。

### Case B：微信发消息被腾讯风控阻断（2026-05-16）

员工反馈：微信经常发消息发不出去，被服务端阻断 / 异地登录提示。

排查——解析微信关键域名 IP：

| 域名 | 关键 IP 段 |
|---|---|
| `short.weixin.qq.com`（发消息长连接） | `43.129.255.x` |
| `long.weixin.qq.com`（长连接） | `129.226.107.x`、`43.129.254.x` |
| `dns.weixin.qq.com`（微信自建 DNS） | `43.129.138/139.x` |
| `szshort.weixin.qq.com`（深圳节点） | `43.155.124.x` |
| `hkshort.weixin.qq.com`（HK 节点） | `43.129/43.159/43.163.x` |
| `mmsns.qpic.cn`、`wx.qlogo.cn` | `43.156.x、101.32.x、129.226.x、150.109.x` |

**这些全是腾讯云海外区（HK/SG/JP/硅谷）IP 段**：`43.129/155/156/159/163`、`129.226`、`101.32`、`150.109`。腾讯有意把微信接入点部分放在腾讯云跨境区做加速。

mihomo geoip 库**不把这些段标为 CN**（注册地真不是 CN，是 HK/SG/JP）。在没有显式 qq.com 规则的模板下，流量路径：

1. 域名分流：rules 段没列 qq.com，**不命中**
2. `GEOIP,CN`：IP 是海外腾讯云，**不命中**
3. 落到 `MATCH,🐟 漏网之鱼` → **走 JP 出口代理**
4. 腾讯长连接服务端看到员工账号从日本 IP 接入 → **异地登录风控 / 消息阻断**

**两个 Case 同根、方向相反**：

| Case | geoip 错判方向 | 落到分流哪一步 | 实际链路 | 后果 |
|---|---|---|---|---|
| LinkedIn | 本该非 CN 被标 CN | `GEOIP,CN` 命中 | 直连出去 | 服务端按本地 IP 跳 `.cn` |
| 微信 | 本该 CN 被标非 CN | `GEOIP,CN` 不命中，落 `MATCH` | 走代理出去 | 服务端按代理 IP 触发异地风控 |

## 根因

两层叠加：

1. **geoip 库精度问题**：mihomo 内置的 MaxMind GeoLite2 / 同类商业库以 ASN 注册信息为主，**只能判断 IP 段在哪个 ASN 注册时填的国家**——
   - 大公司在 CN 注册过的段（LinkedIn 老 PoP）即使早不在中国用，仍然标 CN
   - 腾讯故意宣告在 HK/SG/JP 的段，无论实际服务是给谁用的，都标海外
2. **分流策略依赖 geoip 兜底**：业务真正想表达的是"**这个公司/服务我要走直连还是代理**"，但配置写的是"看 IP 是不是中国"——前者跟域名绑定，后者跟 IP 库的精度绑定，**抽象层错了**。

这是一类系统级反模式：**用 IP 地理位置代理"业务归属"判断**。只要目标公司用了跨境 CDN、anycast、多区接入点，这个代理关系就会断。

## 修复

**核心原则：对**所有"业务归属明确"的国际大厂域名，加显式 `DOMAIN-SUFFIX` 规则，放在 `GEOIP,CN` 之前。** 让分流决策**绑定域名（业务归属）而非 IP（不可靠的地理推测）**。

Clash 模板：

```yaml
rules:
  # 国内服务：强制直连，避免被 geoip 错标为非 CN 走代理被风控
  - DOMAIN-SUFFIX,qq.com,🇨🇳 国内网站
  - DOMAIN-SUFFIX,weixin.qq.com,🇨🇳 国内网站
  - DOMAIN-SUFFIX,wechat.com,🇨🇳 国内网站
  - DOMAIN-SUFFIX,qpic.cn,🇨🇳 国内网站
  - DOMAIN-SUFFIX,qlogo.cn,🇨🇳 国内网站
  - DOMAIN-SUFFIX,tencent.com,🇨🇳 国内网站
  - DOMAIN-SUFFIX,tencent-cloud.com,🇨🇳 国内网站
  # （建议同样补：alipay / dingtalk / bilibili / 微博 等）

  # 国外服务：强制走代理，避免被 geoip 错标为 CN 走直连被服务端按地区跳转
  - DOMAIN-SUFFIX,linkedin.com,🌍 国外网站
  - DOMAIN-SUFFIX,licdn.com,🌍 国外网站
  # （可能还需补：microsoft.com 部分子域、某些 Office 365 端点、Apple 大陆区端点等——按反馈逐步加）

  # 然后才是 GEOIP / MATCH
  - GEOIP,CN,🇨🇳 国内网站
  - MATCH,🐟 漏网之鱼
```

sing-box 同理（注意是放在 `rule_set: geosite-cn` / `geoip-cn` **之前**）：

```json
"rules": [
  {"protocol": "dns", "outbound": "dns-out"},
  {"ip_is_private": true, "outbound": "direct"},
  {"domain_suffix": ["qq.com", "weixin.qq.com", "wechat.com", "qpic.cn", "qlogo.cn", "tencent.com", "tencent-cloud.com"], "outbound": "direct"},
  {"domain_suffix": ["linkedin.com", "licdn.com"], "outbound": "proxy"},
  {"rule_set": "geosite-cn", "outbound": "direct"},
  {"rule_set": "geoip-cn", "outbound": "direct"}
]
```

### 一个局限：客户端用 IP 直连不走 DNS 的场景

微信 / 钉钉等 IM 客户端会在系统级 DNS 失败时**用内置 IP 列表直连**，这种场景下 `DOMAIN-SUFFIX` 规则无效，只能靠 `IP-CIDR` 兜底。如果反馈仍有阻断，补腾讯云海外段的 IP-CIDR 走直连（段多，按需逐步加）：

```yaml
- IP-CIDR,43.129.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,43.155.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,43.156.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,43.159.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,43.163.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,129.226.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,101.32.0.0/16,🇨🇳 国内网站,no-resolve
- IP-CIDR,150.109.0.0/16,🇨🇳 国内网站,no-resolve
```

`no-resolve` 关键——这些 IP-CIDR 规则**不要触发 DNS 反查**，否则匹配速度急剧下降。

## 诊断流程

通用三步：

1. **明确现象**：地理跳转？登录风控？403？只在 X 公司发生？
2. **查客户端连接面板**：mihomo / Clash 客户端的 connections 面板能直接看出每个域名命中了哪条规则、走的哪个出站。截图最直观，特别注意"规则"列写的是 `GEOIP(cn)` / `MATCH` / `DomainSuffix(X)` 之类
3. **反查 IP 归属**：把命中错误的域名解析一下，看 IP 段在 `ipinfo.io` / `ipip.net` 的归属。如果跟你直觉不一致，就是 geoip 库错标了

修复永远是**加显式域名规则**，不是去修 geoip 库。

## 适用范围

- **适用于**：所有用 `GEOIP,CN` + `MATCH` 兜底分流策略的客户端（mihomo / Clash Meta / sing-box / Surge / Quantumult X 等）。geoip 库再准也覆盖不了"大厂把接入点放跨境区"的策略选择
- **不影响**：基于 geosite（按域名归类）的规则集——`geosite-cn` 这种从域名维度直接判断的方式不会有这个问题。但 geosite 库本身也有覆盖不全的情况，对**自家有海外业务的中国厂商**（腾讯云、阿里云国际、字节 TikTok）依然容易错
- **未来如何避免**：架构层把"分流的判断维度"从 IP **整体迁移到域名/业务归属**。对每个明确的业务来源（公司/服务），加一条显式规则；GEOIP 只用于真正未知的兜底场景

## 相关

- 项目源：`vpn-node/scripts/trade-ai-sub.py`（订阅模板里的 rules 段）
- 相关反模式：
  - [mihomo VLESS+Reality 把 server 域名当 SNI](./mihomo-reality-domain-server-becomes-sni.md)
  - [3X-UI 默认 PQ Encryption 静默打挂 mihomo](./3x-ui-default-pq-encryption-breaks-mihomo.md)
- 故障时间：LinkedIn 2026-05-13、微信 2026-05-16
