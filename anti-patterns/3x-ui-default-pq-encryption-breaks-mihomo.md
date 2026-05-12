# 反模式：3X-UI 创建 VLESS inbound 默认开 PQ Encryption，把 mihomo / sing-box 老版本静默打挂

## 一句话结论

3X-UI v2.9.4 创建 VLESS inbound 时默认启用 Xray 25.10+ 的 **VLESS Encryption（mlkem768x25519plus）后量子加密层**，而 mihomo / sing-box 大多数现役版本都不支持。开了之后客户端连不上、且服务端**没日志**（不进 Xray fallback、warning 级别静默），调起来像鬼打墙。

## 场景

- 用 3X-UI（或类似 X-UI fork：xui.one、Sanaeitsoft、Marzban 等都可能跟进）开新 VLESS+Reality inbound
- 客户端是 Clash Verge Rev / ClashX / sing-box for Android / Stash / Shadowrocket（这些都基于 mihomo 或 sing-box 内核）
- 配置看起来"一切正常"，UUID / Reality pubkey / shortId 全对得上，但就是连不上

## 踩坑经历

2026-05-09 自建机场。客户端 Mac（Clash Verge Rev v2.4.7 / mihomo v1.19.21）：

```
$ curl --connect-timeout 15 -x http://127.0.0.1:7897 https://www.google.com -I
HTTP/1.1 200 Connection established       ← Clash HTTP 代理本身握手成功
curl: (35) LibreSSL SSL_connect: SSL_ERROR_SYSCALL  ← 通过代理转发后 TLS 失败
```

服务端 Xray loglevel=warning，故障期间 `journalctl -u x-ui` **零相关日志**。Reality 握手看上去成功（不进 fallback），但应用层 VLESS 帧死掉。

排查死路：检查过 short-id 长度、flow vision、chrome 指纹、mihomo 内核版本——全是无关项。

决定性实验是停掉 3X-UI、用一份干净 config 单独拉起 xray，把 loglevel 设成 debug：

```bash
systemctl stop x-ui   # 同时杀掉它管理的 xray
nohup /usr/local/x-ui/bin/xray-linux-amd64 -c /tmp/xray-test.json > /tmp/xray-test.log 2>&1 &
```

干净 config 的关键差异：`decryption: "none"`，并删掉 `encryption / selectedAuth / testseed`。
客户端 → **0.4s HTTP 200**，调试日志全亮：

```
proxy/vless/inbound: received request for tcp:www.google.com:443
proxy: XtlsFilterTls found tls 1.3! TLS_CHACHA20_POLY1305_SHA256
proxy: CopyRawConn splice
```

锁定根因。

## 根因

3X-UI v2.9.4 创建 VLESS inbound 时往 `settings` 里塞了：

```json
{
  "decryption": "mlkem768x25519plus.native.600s.eEOWUkY-…",
  "encryption": "mlkem768x25519plus.native.0rtt.nw-d-CSq4N9…",
  "selectedAuth": "X25519, not Post-Quantum",
  "testseed": [900, 500, 900, 256]
}
```

这是 **VLESS Encryption（PQ 后量子加密层）**，是 Xray 25.10+ 引入的、**独立于 Reality 的另一层加密**。

- Reality 验证（pubkey + shortId + SNI 探测）依然正常通过 ✓
- 但握手完成后的 VLESS 帧需要再过 PQ 解密层
- mihomo v1.19.21 / sing-box ≤ 1.10 的 VLESS outbound 完全不知道 PQ 这回事，发出来的帧没 PQ 段
- 服务端 PQ 解密失败 → 直接断连，**不进 Reality fallback**（fallback 是给 SNI 不匹配的伪装流量用的），也**不在 warning 级别记日志**
- 客户端拿到的就是 RST，表现为 `SSL_ERROR_SYSCALL` 或 `unexpected EOF`

## 修复

3X-UI 每次重启都从 `/etc/x-ui/x-ui.db` 重建 `/usr/local/x-ui/bin/config.json`，**改文件没用，必须改 db**：

```python
import sqlite3, json
con = sqlite3.connect("/etc/x-ui/x-ui.db")
row = con.execute("SELECT settings FROM inbounds WHERE port=443").fetchone()
s = json.loads(row[0])
s["decryption"] = "none"
for k in ("encryption", "selectedAuth", "testseed"):
    s.pop(k, None)
con.execute("UPDATE inbounds SET settings=? WHERE port=443",
            (json.dumps(s, ensure_ascii=False),))
con.commit()
```

```bash
systemctl restart x-ui
```

或者在 3X-UI Web 面板里编辑 inbound，把"VLESS Encryption / 后量子加密"开关关掉再保存（不同 fork UI 标签略有差异）。

## 验证方法

下次再遇到"Reality 配置全对但客户端 `SSL_ERROR_SYSCALL`"：

1. 把 Xray loglevel 调到 `debug`
   - 3X-UI 管的话改 db 里的 loglevel 字段然后 `systemctl restart x-ui`
   - 或者 `systemctl stop x-ui` 后用独立 config.json 跑 xray
2. 看 debug 日志里是否出现 `decryption` / `mlkem` / `Post-Quantum` 相关词
3. 客户端是 mihomo / sing-box 时第一反应就该怀疑 PQ 层——这两个内核对 Xray 25.x+ 新特性跟进很慢

## 适用范围

- **适用于**：3X-UI v2.9.x（其他 X-UI fork 跟进 Xray 25.10+ 后会出现一样的问题）+ mihomo / sing-box 客户端
- **不影响**：v2rayN（用的就是 Xray 内核，自然支持 PQ）、官方 Xray 客户端
- **未来如何避免**：mihomo / sing-box 加上 PQ 支持后，这个反模式就过期了——但在那之前，**任何用 X-UI 类面板搭机场给 Clash 用户用**的人都该把这一条作为开服 checklist 第一项

## 题外话：为什么这个坑特别难

三个因素叠加在一起：

1. **没日志**：服务端 warning 级别静默，客户端只看到 `SSL_ERROR_SYSCALL`（一个最没信息量的报错）
2. **不是 fallback**：Reality 设计里的 fallback 是用来骗 GFW 探测的，PQ 解密失败不走这条路。新人会想"Reality 不是有 fallback 兜底吗"，但 fallback 这里救不了你
3. **db / config 二元结构**：很多人去改 `config.json`，发现下次重启又变回来，怀疑自己改的位置不对，浪费几小时

记住一个原则：**3X-UI 类面板的所有配置都以 db 为唯一真相源**，config.json 是渲染产物。

## 相关

- 项目源：`vpn-node/notes/2026-05-09-vless-encryption-mihomo-incompat.md`
- Xray PQ Encryption 文档：[XTLS/Xray-core Release 25.10.x](https://github.com/XTLS/Xray-core/releases)
- 故障时间：2026-05-09
