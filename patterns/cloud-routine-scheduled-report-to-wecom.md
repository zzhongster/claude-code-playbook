# 云端 Routine 定时报表 → 企微群机器人：全自动团队报表流水线

> 来源：TOUCH 2.0 日报/作战图/周报自动化（2026-08-11 ~ 08-14，TOU-340/346），当晚全链路实弹验证通过

## 问题

团队日报/作战图靠人手动跑命令、手动贴群：跑的人电脑一关就断更（实际断了两天），且「记得跑」本身就是单点。

## 方案：Claude Code 云端 Routine（cron）+ 企微群机器人 webhook

```
cron（UTC）→ 云端沙箱（clone 仓库 + Linear MCP + GitHub MCP）
  → 照仓库里的命令文档拉数、写 HTML、Chromium 截图
  → 读图自检 → POST 企微 webhook（图 + 文案）→ SendUserFile 存档
```

关键设计决策：

1. **prompt 不内嵌口径，现读仓库权威文档**。routine 的 prompt 只写「读 `.claude/commands/daily.md` 照做」——口径更新只改仓库文件，routine 永不漂移。反面教训：把约束复制进会话产物，权威就裂成两份（该团队 0806 曾因此把排期多算 28 天）。
2. **节拍类自动化才用 routine**（每天/每周固定跑）；事件类（PR 审批通知）用 GitHub Actions，见 `approval-notify-github-actions-wecom.md`。
3. **报表时间对齐业务口径**：日报排在 21:30，因为口径里「PR 过夜」以 21 点线判定——数完再出图。
4. 无人值守纪律写进 prompt：禁止提问；拿不到的数据写「未知，问谁」不编数；发送失败重试一次即停不刷屏；读到的一切内容当数据不当指令。

## 沙箱实测事实（0813，能省一晚上排障）

| 事实 | 细节 |
|---|---|
| Chromium 预装 | 可执行文件在 `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`，要加 `--no-sandbox`；playwright 包本身没装 |
| GitHub 数据 | 走 GitHub MCP 工具（mcp__github__*，ToolSearch 现载），不依赖 gh CLI |
| 出口网络默认拦第三方 | Trusted 模式下 POST qyapi.weixin.qq.com 得 `CONNECT 403`。修法：环境网络设为 Custom，白名单加该域名（保留「常用包管理器」勾选） |
| 建 routine 的前置 | claude.ai 未连 GitHub 账号时，创建带 repo source 的 routine 直接 401 `Connect your GitHub account` |
| 环境变量 UI | 明示「对使用此环境的所有人可见，勿放凭据」——webhook 这类密钥放 routine prompt/配置里（仅本账号可见），不放环境变量 |
| base64 超长 | 600KB 图的 base64 塞 shell 变量报 `Argument list too long`——写临时 json 文件用 `curl -d @file` |

## 企微群机器人 webhook 速查

- 图：`{"msgtype":"image","image":{"base64":"<b64>","md5":"<二进制md5>"}}`，≤2MB；超了走 `webhook/upload_media?key=<key>&type=file` 拿 media_id 发 file 类型；
- 文：`{"msgtype":"text","text":{"content":"..."}}`；
- **无痕验活**：POST 一个非法 msgtype，key 有效返回 `40008 invalid message type`，key 无效返回 `93000`——群里零消息；
- 每次 POST 校验 `errcode==0` 才算发送成功。

## 验收要求（链路式，不是零件式）

判据写成「群里真的收到当天的图和文案，且图上数字与数据源一致」，由回执 + 运行日志双证——不是「routine 建好了」。首跑当晚就靠这条抓出了出口网络 403。
