# 反模式：Clash/mihomo DNS `fallback` 走代理 → url-test 健康检查死锁 → 节点全标 Error

## 一句话结论

fake-ip 模式下，DNS 配 `fallback: [国外 DoH]`（如 `1.1.1.1`/`dns.google`）+ `fallback-filter geoip:CN`，会让**外国域名的解析走代理出去**。而 url-test 做健康检查时要解析测试域名（gstatic 等）——**解析依赖代理、代理正被这次健康检查测试 → 死锁卡 5 秒超时 → 所有节点 delay 失败 → 全标 Error → "自动选择"选不出节点 → 整个订阅"用不起"**。

## 场景

订阅模板里一段"看起来很标准"的 fake-ip DNS 配置：

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  fallback:                              # ← 元凶
    - https://1.1.1.1/dns-query
    - https://dns.google/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN                       # 非 CN 域名 → 交给 fallback 解析
```

意图是好的：国内域名用国内 DNS，国外域名用"干净"的国外 DoH 防污染。但国外 DoH **默认走代理**，于是：

```
url-test 测节点 A 健康度
  → 经节点 A 访问 https://www.gstatic.com/generate_204
    → 先解析 gstatic.com → fallback-filter 判定非 CN → 交给 1.1.1.1 DoH
      → 该 DoH 走代理（节点 A）出去 → 但节点 A 正在被这次健康检查测试、还没就绪
        → 解析等代理、代理等解析 → 死锁 → 5s 超时 → delay 失败 → 节点 Error
```

所有节点同此命运 → 全 Error → 自动选择瘫痪。

## 为什么难定位

- 日常浏览走 **fake-ip**，命中时根本不真解析 → 平时"能用"，bug 隐藏。
- 只有触发**真实解析**（url-test 健康检查、IP 规则匹配、个别 App）才踩 5s 死锁。
- 表现成"时好时坏 / 自动选择失灵 / 偶发卡顿"，一旦被迫依赖 url-test 自动切节点（如主节点 IP 被封）就集中爆发成"全挂"。

## 定位手段（决定性）

mihomo 有 DNS 查询 API，直接量解析耗时：

```bash
curl --unix-socket <mihomo.sock> "http://localhost/dns/query?name=www.google.com&type=A"  # 外国域名
curl --unix-socket <mihomo.sock> "http://localhost/dns/query?name=www.baidu.com&type=A"   # 国内域名
```

中招时：外国域名 **5.01s**（`context deadline exceeded`），国内域名秒回 → 锁定 DNS、且只卡外国域名。
对照一份"能用"的配置（如某商业机场），会发现它**根本不用 fallback**，nameserver 全是国内 DoH。

## 正确做法

fake-ip 场景下，`nameserver` 用**国内 DoH 直接解析所有域名**（包括外国域名；国内 DoH 解析 google 完全够用，且**不走代理、无死锁**），**删掉 `fallback` + `fallback-filter`**：

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
    - 223.5.5.5
    - 119.29.29.29
  # 不要 fallback / fallback-filter
```

## 通用教训

- **节点全标 Error，先怀疑 DNS / 健康检查，别先怀疑节点。** 节点 TCP 能通但 url-test 全失败，十有八九是"解析/测试 url 出不去"。
- **"解析依赖代理 + 代理依赖解析"是经典死锁**。任何让 DNS 解析走代理、而代理就绪又依赖解析的配置都可能踩。
- 隐藏 bug 常在"主路径正常、降级路径才触发"的地方——这个 bug 平时被 fake-ip 掩盖，只在 url-test/规则匹配时现形。

（来源：vpn-node 项目 2026-06-24。原始 notes：`notes/2026-06-24-dns-fallback-deadlock.md`）
