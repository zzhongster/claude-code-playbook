# 反模式：REALITY 伪装回落域名会随大厂升级 TLS 而"过期"，打挂全员握手

## 一句话结论

VLESS+REALITY 的伪装回落目标（`target` / `dest` / `serverNames`，如 `www.microsoft.com`）不是一劳永逸的配置。**当这个大厂域名升级了 TLS 握手行为（post-quantum keyshare / ECH / ServerHello 结构变化）后，REALITY 的"偷证书"中继就处理不了它了**，服务端能取到证书却完不成握手，把每个客户端降级成透明回落踢掉 → **全员直连节点集体挂掉**。现象极像"IP 被封"，但换个伪装目标就立刻恢复。

## 背景：REALITY 靠"偷"一个真网站的 TLS 握手

REALITY 的抗封锁原理：服务端不自己签证书，而是实时连接一个真实大厂网站（`target`，如 `www.microsoft.com:443`），把它的 TLS ServerHello + 证书"偷"过来伪装。GFW 主动探测时看到的就是一个真·微软 TLS 会话，看不出破绽。

这个机制的隐藏依赖：**服务端必须能把 target 的 TLS 握手完整中继给客户端**。target 的握手长什么样，REALITY 就得能转发什么样。一旦 target 换了握手方式，中继可能就断在半路。

## 踩坑经历

2026-07-08。自建机场用户报"直连是不是完全不能用了"。现象：mihomo 里 **JP-Reality 节点 `alive=False`，delay 测试一律 `An error occurred in the delay test`**，url-test 自动选择长期停在 CF-WS 兜底线路上（全员被迫走兜底，直连快线彻底用不了）。CF-WS（VLESS+WS+TLS 走 CF 中转）完全正常，**只有 REALITY 直连挂**。

第一反应很容易误判成"datacenter IP 又被 GFW 软封了"（该项目 2026-06 刚因软封迁过一次机房）。但这次是配置问题，差点又冲动去迁机房。

## 排查：逐层排除，别猜（关键是 localhost 自测）

| 层 | 手段 | 结果 |
|---|---|---|
| 网络层 | `ping -c20 <vps>` | 0% 丢包，75~108ms → 网络没问题 |
| TCP | VPS 上 `tcpdump` 抓 :443 | 客户端完整三次握手 + 发出 ClientHello，服务器回包 → **不是 IP 被封 / 封端口** |
| 服务 | `systemctl is-active x-ui`、`ss -tlnp` | active，443 正常 listen |
| 密钥 | `xray x25519 -i <privkey>` 派生公钥 | 与 db、订阅完全一致 → 不是密钥失配 |
| **REALITY 本机自测** | localhost 起 client 连生产 :443 | **握手失败**（http 000） |
| **全新密钥对自测** | 新 keypair + 新 UUID，localhost server+client | **仍失败** → 彻底排除密钥 |
| **只换回落 target** | 同一套新密钥，只改 target/SNI | microsoft→**000**；apple→**204**；cloudflare→**204** |

> **决定性一步 = localhost 自测**。在 VPS 本机起一对 xray server+client 互连（绕开 GFW、绕开公网、绕开客户端网络），把问题锁死在"服务端配置"而非"IP 被封"。再用"全新密钥对 + 只变单一变量（target）"的对照实验，一步定位到唯一变量就是伪装目标域。

## 决定性证据 & 根因

服务端 REALITY debug 日志（`realitySettings.show=true`，target=microsoft）：

```
REALITY remoteAddr: ...  Server Hello: 127
REALITY remoteAddr: ...  Change Cipher Spec: 6
REALITY remoteAddr: ...  Encrypted Extensions: 51
REALITY remoteAddr: ...  Certificate: 8273          ← 从 microsoft 取到了完整 TLS1.3 证书
REALITY remoteAddr: ...  hs.c.isHandshakeComplete.Load(): false
[Info] transport/internet/tcp: REALITY: processed invalid connection ...: handshake did not complete successfully
```

服务器**能**从 `www.microsoft.com` 取到证书（8273 字节全套 TLS1.3），但握手最终 `isHandshakeComplete: false` → **鉴权/劫持没完成，降级成透明回落**，客户端拿到的是真·微软的 TLS 会话，vless 用不了 → 被踢。

而 VPS 直连 `www.microsoft.com:443` 本身完全正常（`curl -I` tls 44ms / http 200）。**问题不在可达性，在它作为 REALITY 偷证书目标的 TLS 握手已不兼容**——最可能是微软给 `www.microsoft.com` 开了 **post-quantum TLS（X25519MLKEM768 keyshare）** 或改了 ServerHello 结构，xray 26.4.25 的 REALITY 中继处理不了这种握手。

旁证：三次 localhost 自测耗时惊人一致（2.02 / 2.89 / 2.89s 后 fail），符合"卡在与 target 的中继握手上超时"而非"瞬间拒绝"。

换 `www.apple.com` / `www.cloudflare.com` 做 target 立即恢复（204），证明 **REALITY / xray / 密钥 / 网络全部健康，唯一坏的变量就是 target 域**。

## 修复

服务端把伪装目标换成一个 TLS 行为保守、还没升级 PQ 的大厂域：

```python
# /etc/x-ui/x-ui.db 的 inbounds.stream_settings.realitySettings
rs["target"]      = "www.apple.com:443"      # 原 www.microsoft.com:443
rs["serverNames"] = ["www.apple.com"]        # 原 ["www.microsoft.com"]
# 然后 systemctl restart x-ui（3X-UI 会从 db 重建 config.json）
```

验证（生产 :443 本机自测，绕开公网）：

```bash
# 用 db 里的真实 uuid/pbk/sid 起一个 localhost client 连 127.0.0.1:443
curl -s --max-time 10 --socks5 127.0.0.1:<port> \
  -o /dev/null -w "http=%{http_code} time=%{time_total}s\n" \
  http://www.gstatic.com/generate_204
# 修好后：http=204 time≈0.01s（秒握手）
```

### ⚠️ 换 SNI 后必做：通知全员重拉订阅

伪装 SNI 变了（microsoft→apple），**没重拉订阅的客户端 REALITY 会一直握手失败**（自动停在兜底线路，业务不断但直连快线用不了）。这条和"换 IP 后必做"是同一级纪律。若订阅服务是实时从 db 读 `serverNames[0]` 渲染 SNI（如 `trade-ai-sub.py`），则**改 db + 客户端点一次"更新订阅"即可自动对齐，不用改脚本模板**。

## 如何提前避免 / 选目标的原则

- **伪装目标要挑 TLS 行为保守、变动少的域**。越是"技术领先"的大厂（Google / Microsoft / Cloudflare 自己的门户）越可能率先上 PQ TLS、ECH 这些新东西，反而不稳。
- 本次实测可用的备选池（2026-07）：`www.apple.com`、`www.cloudflare.com`。挑的时候优先满足：支持 TLS1.3、非 CDN 频繁变动、`curl -I` 稳定 200、且该域在国内直连不被墙（否则 GFW 一看 SNI 就知道是假的）。
- **把"换伪装目标"当成一项常规运维预案**，而不是灾难。REALITY 目标会周期性"过期"，备好 2~3 个候选、知道怎么一条 SQL + restart 切换，比每次都从头排查快得多。
- 排查工具箱：
  - `xray x25519 -i <priv>` 验证公钥与私钥匹配
  - localhost 起一对 server/client，**只变单一变量**做对照（target / 密钥 / flow 各测一遍）
  - `realitySettings.show=true` 看握手分步日志（Server Hello / Certificate / isHandshakeComplete）
  - `curl -I https://<target>` 确认 VPS 到 target 可达且是普通 200（可达 ≠ 能当 REALITY 目标）

## 判据：REALITY 全员挂，先分清三层

**"节点全 Error / `alive=False`，别第一反应就是 IP 被封。"** 按这个顺序切：

1. **DNS / 健康检查层**？（见 [mihomo DNS fallback 死锁](./mihomo-dns-fallback-deadlock-healthcheck.md)）—— 节点其实好的，是健康检查自己死锁。
2. **网络 / IP 层**？—— `ping` 丢包率 + `tcpdump` 看 SYN/RST。软封是概率丢包，硬封是 RST 注入或端口不通。
3. **服务端配置层**？—— **localhost 自测**一步绕开前两层。密钥失配、`decryption` 非 none、或本条：**伪装目标失效**。

## 适用范围

- **适用于**：所有 VLESS+REALITY 部署（Xray 服务端 / sing-box 服务端同理），任何用大厂域名做 `target`/`dest` 偷证书的场景。
- **触发条件**：伪装目标域升级了 TLS 握手（PQ keyshare、ECH、握手结构变化），且当前 xray/sing-box 版本的 REALITY 中继跟不上。升级 xray 有时也能修（新版跟进了新握手），但换目标最快最稳。
- **不是这条的情况**：
  - 单个客户端连不上而非全员 → 更可能是那个客户端的 server 写了域名（见 [mihomo Reality server 当 SNI](./mihomo-reality-domain-server-becomes-sni.md)）或密钥/decryption 问题。
  - 能连上但不稳/打不开网页 + 多线路都差 + 服务端全健康 → 才是 IP 软封，该迁机房。

## 相关

- 项目源：`vpn-node/notes/2026-07-08-reality-microsoft-target-broken.md`
- 相关反模式：
  - [3X-UI 默认 PQ Encryption 静默打挂 mihomo](./3x-ui-default-pq-encryption-breaks-mihomo.md)（同样是 PQ 密码学新特性打挂老实现，只不过那次在 client 加密协商，这次在伪装 target 的 TLS）
  - [mihomo Reality 把 server 域名当 SNI](./mihomo-reality-domain-server-becomes-sni.md)
  - [mihomo DNS fallback 死锁误报节点全挂](./mihomo-dns-fallback-deadlock-healthcheck.md)
- 故障时间：2026-07-08
