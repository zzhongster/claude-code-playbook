# 反模式：用「独立 IP 数」判密钥分发——把 CI runner、探针和移动办公全标成疑似泄露

## 一句话结论

「一把 key 一周从多少个 IP 打过来」回答的是**去过多少地方**，而"密钥被分发"要问的是**是否同时在多处使用**。拿前者当判据，两台生产机的周报各标了 2 把 key，**全是误报**：GitHub Actions runner、内部探针、销售的移动办公。误报一周一次，人很快就不看它了——而这道门本来是为了那个真的把 key 转给别人的人。

## 场景

- 按 key 计费/限流的 API，账本每笔带来源 IP，想检测"一把个人 key 被分给多人用"
- 报表周期性跑（cron），按"独立 IP 数 > 阈值"标记并用退出码报警
- 使用方里混着：CI 流水线（每次跑换一台云主机）、多机部署的健康探针、用手机热点/家里/公司/客户现场轮换上网的销售

## 踩坑经历

trade-agent-data（2026-09-08）。`ip-report` 上线三周后第一次有人认真看主账本的输出：

| key | 独立 IP | 真相 |
|---|---|---|
| `touch-dev` | 16（全是 Azure 段） | 下游的契约守卫跑在 GitHub Actions hosted runner 上，每次一台新机器 |
| `internal-ops` | 4 | 两台生产机的冒烟探针 + 办公网 + 本机回环 |
| `wangyang`（另一台机器的账本） | 15（含多个移动运营商段） | 销售移动办公，先后换网 |
| `lujianlin` | 5 | 同上 |

四把全部"⚠ 疑似被分发"，退出码 3。没有一把是分发。

## 为什么会错

选判据时拿了**容易查的观测量**（`COUNT(DISTINCT ip)` 一条 SQL）替代**真正要回答的问题**（同一时刻是否多处在用）。两者在"一个人先后换网"与"一把 key 同时被三个人用"上给出**相同的数字**，判据没有区分力。

同族：[指标绿的原因不是你以为的](green-metric-measures-wrong-mechanism.md)、[没有区分力的证据](evidence-without-discriminating-power.md)。

## 正确做法

三层，缺一层都会回到周周误报：

1. **按档位豁免，不按名字硬编码**。internal/CI 档的 key 天然多 IP（runner、探针），报表默认不计入，`--include-internal` 恢复。按名字写 `if name == "touch-dev"` 是把一个类别编码成一个名字，第二个同类出现时没人记得扩它。
2. **人工核实过的正常多 IP 用 `--exempt a,b` 点名**，写在 cron 命令里可追溯；仍列出并标「已豁免」，看得见、过期了能发现。不要放宽阈值——放宽会把真分发一起放过。
3. **加一个有区分力的观测量：并发窗口**——同一 10 分钟窗口内出现 ≥2 个不同 IP 的窗口数。移动办公是先后换网（≈0），分发是同时多处在用（>0）。SQLite 一句：

   ```sql
   SELECT name, COUNT(*) FROM (
     SELECT k.name, strftime('%Y-%m-%d %H', l.ts) || (CAST(strftime('%M', l.ts) AS INTEGER)/10) AS win,
            COUNT(DISTINCT l.ip) AS n
     FROM ledger l JOIN keys k ON l.key_hash = k.key_hash
     WHERE l.ip IS NOT NULL AND l.ts >= datetime('now', '-7 days')
     GROUP BY k.name, win HAVING n >= 2) GROUP BY name;
   ```

   主账本实测：客户 key **全部 0**，探针 key 18（两台机器本来就并发），CI key 1。这一列只做**给人核实用的线索**，不做自动裁决——名称/行为启发式只标记不裁决，是同一套账本上另一条纪律（按名称相似度合并公司那次的教训）。

顺带一条：这类报表**只写 cron 日志等于没报**。同一份周报在旧机器上跑了三周没人打开过；改成退出码 3 时经企微私信 `--notify`。合作方那边"守卫连红 15 天 / 5 天没人看"两次复发，都是同一个形态：日志不是通知。

## 适用范围

- 适用于：任何"共享凭证/账号被多人使用"的检测——API key、SaaS 子账号、VPN 账号；以及更一般的"用基数（去过多少处）替代同时性（是否并发）"的判据
- 不适用于：来源 IP 本身不可信（全走 NAT/代理出口）的场景——那时连并发窗口也失去区分力，要换设备指纹或会话 ID

## 相关

- [green-metric-measures-wrong-mechanism](green-metric-measures-wrong-mechanism.md)
- [evidence-without-discriminating-power](evidence-without-discriminating-power.md)
- [scheduled-job-only-alerts-on-failure](scheduled-job-only-alerts-on-failure.md)（定时任务的另一类失效：根本没跑 / 跑了没人看）
