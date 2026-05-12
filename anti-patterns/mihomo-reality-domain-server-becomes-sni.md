# 反模式：mihomo VLESS+Reality 把 `server` 字段当 SNI 用，绕过 `servername`

## 一句话结论

mihomo（Clash Meta 内核）在 VLESS+Reality 场景下，当 `server` 字段是**域名**时会把它当成 TLS SNI 发出去，**无视 `servername` / `reality-opts.public-key` 配套写的伪装域名**。结果是 Reality 服务端收到错的 SNI、握手失败、连不上。

## 场景

你照着官方 mihomo 文档写 VLESS+Reality outbound：

```yaml
proxies:
  - name: jp-reality
    type: vless
    server: vps.example.com        # ← 用域名（订阅服务、SSL 证书都方便）
    port: 443
    uuid: xxxx-xxxx-…
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    client-fingerprint: chrome
    servername: www.microsoft.com  # ← 期望 SNI 是这个伪装域名
    reality-opts:
      public-key: <pbk>
      short-id: <sid>
```

看起来一切都对。但只要 `server` 是域名（不是 IPv4 字面量），连接就会失败。

## 踩坑经历

2026-05-09。自建机场搬到域名 `yicha-v.trade-ai-sgp.com` 之后整套连不上了，
之前用 IP `192.243.121.152` 一切正常。

排查路径：客户端 Clash Verge Rev v2.4.7 / mihomo v1.19.21，把 mihomo log-level 调到 debug：

```
[DEBUG] [TLS] REALITY: server name mismatch: yicha-v.trade-ai-sgp.com
[DEBUG] [VLESS] handshake failed: invalid response
```

服务端 Reality 期望 SNI = `www.microsoft.com`（伪装域名），实际收到的是 `yicha-v.trade-ai-sgp.com`（订阅域名）——`servername` 字段被忽略了。

唯一差异：之前手写 YAML 用 IP `192.243.121.152`，新订阅生成的 YAML 用域名 `yicha-v.trade-ai-sgp.com`。
切回 IP 立刻好。

## 根因

mihomo 的 VLESS outbound 在初始化 TLS 配置时，逻辑大致是：

```go
if isDomain(server) {
    tlsConfig.ServerName = server   // ← 这里直接覆盖了
} else {
    tlsConfig.ServerName = servername
}
```

意图大概是"给普通 TLS 节点提供 SNI 默认值"，但对 Reality 来说是灾难——Reality 的核心机制就是**用一个伪装域名（如 microsoft.com）做 SNI 让 GFW 看不出来**，`server` 字段只是个连接目标地址，不应该参与 SNI 决策。

这个行为在 mihomo issue 区有报告但没被当成 bug 修，因为：
- 改了会破坏现有的"server 是域名时自动设 SNI"的普通 TLS 用户
- Reality 用户只要 `server` 写 IP 就没事
- "你按文档写 `servername` 应该被尊重"——逻辑上对，实现上没有

## 修复

**订阅 YAML 里 `server:` 字段强制使用 IP 字面量**，订阅 URL / 证书 / 域名解析这些层照常用域名：

```yaml
proxies:
  - name: jp-reality
    type: vless
    server: 192.243.121.152        # ✅ IP 字面量
    port: 443
    servername: www.microsoft.com  # ✅ 现在不会被覆盖了
    …
```

如果用 3X-UI 自动生成订阅，要去 db 里设：

```python
# /etc/x-ui/x-ui.db 的 inbounds.stream_settings.externalProxy
[{
  "forceTls": "same",
  "dest": "192.243.121.152",   # ← 这里写 IP，不是域名
  "port": 443,
  "remark": ""
}]
```

3X-UI 生成订阅时会读这个 `dest` 字段塞进 `server:`。

> ⚠️ 这个机场如果用了自建订阅服务（如 `vpn-node/scripts/trade-ai-sub.py`），脚本也是从 `externalProxy.dest` 读 server 的，同样规则适用。

## 验证方法

故障复现 + 修复验证用 mihomo debug 日志：

```bash
# 改客户端配置开 debug
# Clash Verge: 设置 → 日志等级 → debug
# 或 mihomo CLI: config.yaml 里 log-level: debug

# 在客户端日志里搜 REALITY
tail -f ~/Library/Logs/clash-verge/<date>.log | grep -i reality
```

- **错误（server=域名）**：`REALITY: server name mismatch: <你的订阅域名>`
- **正确（server=IP）**：`REALITY: handshake success`，后续看到 `XtlsFilterTls found tls 1.3`

## 适用范围

- **适用于**：mihomo / Clash Meta 全系（v1.18 ~ v1.19+ 已确认）+ VLESS+Reality 组合
- **不影响**：
  - sing-box（独立内核，自己的 SNI 处理逻辑，没有这个 bug）
  - 普通 VLESS+TLS（确实应该用 server 域名做 SNI，符合预期）
  - 用 IP 做 `server` 的所有场景（IP 没法做 SNI，自然不会覆盖）
- **未来如何避免**：mihomo 修了这个 bug 之后这条会过期。在那之前，凡是 mihomo+Reality 都按"server 必须是 IP"对待

## 题外话：订阅服务能用域名吗

能，而且**应该**用。把"用什么地址访问订阅 URL"和"VLESS YAML 里 server 字段写什么"分开：

| 层 | 用什么 | 为什么 |
|---|---|---|
| 订阅 URL (`https://xxx.com:2096/clash/...`) | 域名 | 配 SSL 证书方便，看起来正常，IP 变了不用通知员工 |
| 域名 A 记录 | 解析到 VPS IP | 标准做法 |
| YAML 内部 `server:` | **VPS IP 字面量** | 绕开 mihomo Reality SNI bug |
| YAML 内部 `servername:` | 伪装域名（microsoft.com 等） | Reality 协议要求 |

不是"必须放弃域名"，而是**在订阅产物里把 server 渲染成 IP**。订阅服务模板里这样写：

```yaml
proxies:
  - server: {server_ip}      # 模板渲染时从 db / 配置读 IP
    servername: {sni}        # 伪装域名
```

`vpn-node` 项目的 `trade-ai-sub.py` 就是按这个原则实现的。

## 相关

- 项目源：`vpn-node/notes/2026-05-09-session-summary.md`（第 5 节）
- 相关反模式：[3X-UI 默认 PQ Encryption 静默打挂 mihomo](./3x-ui-default-pq-encryption-breaks-mihomo.md)
- 故障时间：2026-05-09
