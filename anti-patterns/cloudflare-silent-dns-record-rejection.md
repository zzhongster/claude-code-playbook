# 反模式：把 DNS 控制面的"写入成功"当成解析生效

## 一句话结论

Cloudflare 对某些记录名（实测：**含重复标签**的名字如 `send.send.example.com`）会**静默丢弃**——API 返回 `success: true` 并给出 record id、Dashboard 正常显示该行、GET 也读得回，**但区文件里根本不写**，权威 NS 持续返回 NXDOMAIN。控制面的"成功"和数据面的"生效"是两件事，中间没有任何报错告诉你。

## 场景

给第三方服务（Resend / SendGrid / Postmark 等）配发件域 DNS。按后台给的清单把 SPF / DKIM / MX 写进 Cloudflare，UI 里三条都在、内容一字不差，回去点"验证"——**一直不通过**。

## 现场

2026-08-01，给 `trade-ai-sgp.com` 配 Resend 事务邮件发件域。发件域选了 `send.trade-ai-sgp.com`，**Resend 会在发件域前再套一层 `send.` 作为 MAIL FROM**，于是要求的记录名变成 `send.send.trade-ai-sgp.com`。

三条记录一批写入，API 全部返回成功：

```json
[{"type":"TXT","name":"resend._domainkey.send.trade-ai-sgp.com","ok":true,"id":"b3be..."},
 {"type":"MX", "name":"send.send.trade-ai-sgp.com","ok":true,"id":"9592..."},
 {"type":"TXT","name":"send.send.trade-ai-sgp.com","ok":true,"id":"4a95..."}]
```

实际解析结果：

| 记录 | 权威 NS 响应 |
|---|---|
| `resend._domainkey.send.trade-ai-sgp.com` | ✅ NOERROR，正常返回 |
| `send.send.trade-ai-sgp.com` TXT | ❌ **NXDOMAIN** |
| `send.send.trade-ai-sgp.com` MX | ❌ **NXDOMAIN** |

注意是 **NXDOMAIN 而不是 NODATA**——权威服务器认为这个名字**整个节点都不存在**。同一批、同一个 API 调用路径写进去的三条，第一条生效，后两条凭空消失。

Dashboard 里这三条一直好端端列在记录表里，`15 / 200 条`，没有任何警告标记。

## 排查过程（这些都不是原因，写下来是为了让你少走）

| 假设 | 验证方式 | 结果 |
|---|---|---|
| DNS 传播延迟 | 轮询 10 分钟，查两台权威 NS + 1.1.1.1 + 8.8.8.8 | ❌ 四个解析器一致 NXDOMAIN |
| 同名 MX 与 TXT 冲突 | 删掉 MX，只留 TXT | ❌ TXT 仍不解析 |
| API 传 FQDN 被二次拼接 | 改用相对名 `send.send` 重建 | ❌ 同样不解析 |
| 记录需要"触发"才发布 | PATCH 改 TTL 触发重写 | ❌ 无效 |
| 记录没真的存进去 | GET 单条记录 | ❌ 存着呢，内容完全正确 |

## 定位方法：探针二分

猜不动的时候，**别再对着坏记录反复试，去建几个对照记录**。30 秒锁定根因：

```bash
# 三个探针，每次只改一个变量
zztest.send.trade-ai-sgp.com   TXT  →  ✅ 20 秒生效   # 排除"这一层子域有问题"
send.zzprobe.trade-ai-sgp.com  TXT  →  ✅ 生效        # 排除"以 send 开头有问题"
send.mail.trade-ai-sgp.com     TXT  →  ✅ 生效        # 排除"send 作为首标签有问题"
send.send.trade-ai-sgp.com     TXT  →  ❌ NXDOMAIN    # 只有重复标签不行
```

结论一目了然：**根因是重复标签，不是 `send` 这个词，也不是子域层数。**

这个方法对任何"配置写进去了但不生效"的问题都通用——**构造最小差异的对照组，让变量自己暴露**，比读文档和猜实现快得多。

## 正确做法

**1. 写完必须 `dig` 验证，不能只看 API 返回或 UI 显示。**

```bash
dig +short TXT _your_record.example.com @<权威NS>
```

直接问权威 NS（`dig +short NS example.com` 拿到），绕开所有缓存。控制面说什么不算数，权威服务器返回什么才算数。

**2. 避开重复标签。**如果第三方服务会在你的域名前自动加前缀（Resend 加 `send.`、部分服务加 `mail.` 或 `bounce.`），**选发件域时就避开同名**：

| 发件域 | 服务商生成的 MAIL FROM | 结果 |
|---|---|---|
| `send.example.com` | `send.send.example.com` | ❌ |
| `mail.example.com` | `send.mail.example.com` | ✅ |

**3. NXDOMAIN 是强信号。**排查时先分清是 NXDOMAIN 还是 NODATA：

- **NODATA**（NOERROR + 0 answer）= 名字存在，只是没有这个类型的记录 → 往"记录类型/内容写错了"方向查；
- **NXDOMAIN** = 名字整个不存在 → 往"记录压根没进区文件"方向查，别再纠结内容对不对。

## 更一般的教训

这不是 Cloudflare 独有的问题。**任何"控制面 API + 数据面执行"的系统都可能这样**：API 校验通过、状态存进控制面数据库、返回 200，但下发到数据面时被某条规则挡掉，而这条链路上没有反向通知机制。

所以凡是"改配置"类操作，验收都要落在**数据面的可观测结果**上，而不是控制面的返回值。参见 [split-config-check-from-e2e-verification](../patterns/split-config-check-from-e2e-verification.md)。

## 适用范围

- 适用于：Cloudflare DNS 及任何提供"写入 API + 独立执行层"的基础设施服务；
- 不适用于：本地配置文件类改动（改完就是最终状态，没有下发环节）。

## 相关

- [split-config-check-from-e2e-verification](../patterns/split-config-check-from-e2e-verification.md) — 配置层确认与端到端验证要分开记账
