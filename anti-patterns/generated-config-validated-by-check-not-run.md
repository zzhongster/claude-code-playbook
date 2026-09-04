# 反模式：给自动更新的客户端下发生成配置，只用静态 `check` 验证

## 一句话结论

`<工具> check` 通过 ≠ 能启动，更 ≠ 在员工手机上的那个版本能启动。给会自动升级的客户端（sing-box / Clash / 任何"拉远程配置"的 App）下发**服务端生成**的配置，验证必须是：**用最新版 + 仍在野的最低版两个真实二进制，把配置真跑起来、真发几个请求**。少一环，坏法都是"全绿上线、用户报错"。

## 场景

自建服务按模板生成客户端配置（订阅服务、agent 配置分发、IaC 渲染出的运行时配置……），客户端是别人手里会自己升级的 App。典型心态：

- "模板半年没动过，一直好用" —— 模板没动，**解析它的客户端在动**
- "改完跑了 `sing-box check`，全绿" —— check 只做解析 + 构造，很多错在 `Start()` 才炸
- "文档说新字段是 X，直接换成 X" —— 新字段只有最新版认识，老版本严格解析、遇未知字段直接拒载

## 踩坑经历（2026-09-04，vpn-node）

用户更新 iOS sing-box 后导入订阅报：

```
decode config: dns.servers[0]: legacy DNS server formats are deprecated in sing-box 1.12.0
and removed in sing-box 1.14.0
```

模板是 1.10 时代写的（`"address": "https://1.1.1.1/dns-query"`），1.12 弃用、1.14 删除。弃用期两个小版本（约一年），**警告只打在客户端日志里**，服务端和用户谁都没看到，直到删除那天一起爆。

修复过程又踩了三下，每一下都是"验证方法不够真"：

| 轮次 | 做了什么 | 验证 | 结果 |
|---|---|---|---|
| 1 | `dns.servers` 换新格式 | 1.14 `check` 全绿 → **部署上线** | 本地真跑才发现启动 FATAL：`start dns/https[local-dns]: detour to an empty direct outbound makes no sense`。`check` 查不出，只有 `run` 炸 |
| 2 | 去掉 `detour:"direct"`，本地真跑 | 起来了但下载 rule-set 时 `EOF` | 一度以为 CF 中转坏了。真因：测试配置用的**假 UUID**，服务端 Xray 直接掐连接。换真 UUID 全通 |
| 3 | 1.14 真跑冒出新 WARN：`download_detour` 1.14 弃用、1.16 删除，替代 `http_client` | 查文档 | `http_client` 是 1.14 才有的字段，**现在换上去会把 1.12/1.13 的客户端弄坏**（严格解析）。只能记成到期项，等全员 ≥1.14 再迁 |

另外 1.12 `check` 旧配置有**两条** WARN（legacy DNS servers + outbound DNS rule item），只修第一条的话 1.14 照样拒载——弃用警告要**全部**清零，不是修到报错消失。

## 根因

三个结构性原因叠在一起：

1. **静态校验与启动语义是两层。** `check` = 反序列化 + 构造对象；引用合法性、"这个组合没意义"之类的判断在 `Start()`。配置工具几乎都这样（sing-box、nginx `-t` 对 upstream 的可达性、k8s dry-run 对 admission 之外的逻辑……）。
2. **弃用→删除的时间窗只对客户端可见。** 生成方永远看不到自己产物在别人机器上打出的 WARN；用户也不看日志。等报错传回来时已经是"删除"那一天。
3. **兼容窗口是双边的。** 最新版拒绝旧字段，老版本拒绝新字段（严格解析，未知字段 = 拒载）。可用写法 = 两者交集，且交集随时间移动。"照最新文档写"和"一直不动"都会出界。

## 修复 / 正确做法

**1. 验证脚本 = check + 真跑 + 真请求，且用真凭证。**
把需要特权的入站（tun）换成 `127.0.0.1` 的 mixed/socks 入站，本机拉起来，curl 走一遍代理路径和直连路径：

```python
cfg["inbounds"] = [{"type": "mixed", "listen": "127.0.0.1", "listen_port": 21080}]
subprocess.Popen([SB, "run", "-c", runcfg])          # 等端口开，等不到就 FAIL 并打日志
curl -x socks5h://127.0.0.1:21080 https://www.gstatic.com/generate_204   # 代理路径
curl -x socks5h://127.0.0.1:21080 https://www.baidu.com/                 # 直连路径
```

凭证要真的（直接 `--url` 拉线上下发的配置最省事）。假凭证下服务端行为是"掐连接"，客户端侧表现成 `EOF`/超时，**和传输层故障同形**，会把排查引到错误的组件上。

**2. 二进制矩阵：最新版 + 仍在野的最低版，各跑一遍。**
最新版负责暴露"下一个要删的"，最低版负责保证"现在没人被踢出去"。两份都绿才算过。

**3. 把 WARN 当 error 处理。**
弃用警告出现的那一刻就是迁移窗口的起点。脚本里把日志中的 `WARN/FATAL/ERROR` 抓出来打印；出现即建到期项（本例：`download_detour` → `http_client`，1.16 到期）。

**4. 迁移目标受最低版约束，不受文档约束。**
替代字段在最低版认不认？不认就不能换，先催升级、再迁；不要两边一起赌。

**5. 服务端记一下客户端 UA。**
本例服务为了防扫描器刷日志把 `log_message` 静默了，副作用是**看不到员工版本分布**，"最低版是哪个"只能猜。至少把 UA 里的版本号聚合记下来。

## 验证方法

```bash
# 二进制
gh release download v1.14.0 -R SagerNet/sing-box -p 'sing-box-1.14.0-darwin-arm64.tar.gz'
gh release download v1.12.0 -R SagerNet/sing-box -p 'sing-box-1.12.0-darwin-arm64.tar.gz'

# 对线上订阅 URL 直接测（vpn-node/scripts/singbox-smoke-test.py）
python3 scripts/singbox-smoke-test.py --bin ./sing-box-1.14.0 --url https://host:2096/singbox/<subId>
python3 scripts/singbox-smoke-test.py --bin ./sing-box-1.12.0 --url https://host:2096/singbox/<subId>
```

- **修复前**：1.14 `check` 就 FATAL（和手机同一句报错）；1.12 `check` 两条 WARN
- **第一版修复**：两版 `check` 全绿，1.14 `run` FATAL —— 这就是本条反模式的核心证据
- **修复后**：两版 check 干净、run 起来 1.6s、4 条 curl 全通；1.14 只剩 `download_detour` 一条 WARN（已登记到期项）

## 适用范围

- **适用于**：任何"服务端渲染配置 → 客户端自己拉、自己升级"的分发链路：代理订阅、agent/exporter 配置、feature-flag 规则文件、IaC 渲染的运行时配置
- **同理适用于**：CLI 工具的 `--dry-run`/`check`/`validate`/`lint` 子命令——它们验证的层次要看清楚，别默认等于"能跑"
- **不适用于**：客户端版本你自己控制、和服务端一起发版的场景（那时单版本集成测试就够）

## 相关

- [测试全绿，但拆掉修复它还是全绿](guard-passes-without-the-fix.md)：同族——"验证存在"与"验证有效"在绿勾里同形
- [持久化状态跨越部署边界](persisted-state-crosses-deploy-boundary.md)：同族——新旧两版代码通过一份数据通信，单版本测试结构上盖不住
- [SEO 基建只验本地产物不验线上](seo-verified-in-build-never-in-production.md)：同族——验收只认线上 URL
- 项目源：`vpn-node/scripts/trade-ai-sub.py`（`build_singbox`）、`vpn-node/scripts/singbox-smoke-test.py`
- 事故笔记：`vpn-node/notes/2026-09-04-singbox-114-legacy-dns-format-removed.md`
- sing-box 迁移文档：https://sing-box.sagernet.org/migration/
