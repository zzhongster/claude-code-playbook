# 给 GUI 客户端接远程 http MCP 的四个静默坑（npx 路径 + --allow-http + 僵尸桥 + 僵尸堆积）

> 把一个远程 streamable-http MCP server 接进 Claude Desktop / 腾讯 WorkBuddy 这类 **GUI 客户端**时，配置"看起来完全正确"但客户端里工具就是不出现——三个非报错、非直觉的坑，都不在文档显眼处。

## 场景

自建 MCP server 跑在云服务器上（streamable-http，端口对外开、带 token 鉴权），端点 `curl` 实测能通、返 307/200。GUI 客户端用官方推荐的 `mcp-remote` 桥接（stdio↔http 代理），配置写成：

```json
"trade-data": {
  "command": "npx",
  "args": ["-y", "mcp-remote", "http://1.2.3.4:8100/mcp",
           "--header", "Authorization: Bearer <token>"]
}
```

重启客户端 → trade-data 不出现，或报个含糊的 "server failed to start"。schema、端点、token 全对，卡在两个隐蔽点。

## 坑 1：`command: "npx"` 裸命令 → `spawn npx ENOENT`

**Claude Desktop（macOS app）启动子进程时用的是精简 PATH**（`/usr/bin:/bin:/usr/sbin:/sbin`），**不含 Homebrew 的 `/opt/homebrew/bin`、nvm、asdf 等**。所以你终端能跑的 `npx`，客户端里 `spawn npx ENOENT` 找不到。

**修**：`command` 用**绝对路径**（`which npx` 查）：
```json
"command": "/opt/homebrew/bin/npx"
```
（这坑对所有"GUI app 启动的 stdio MCP + 依赖 PATH 里的解释器/工具"通用：node/python/uvx 同理，一律绝对路径。）

## 坑 1.5：绝对路径 npx 也不够——npx 是脚本，运行时还要靠 PATH 找 node

(2026-07-09 实测) 坑 1 修成 `command: "/opt/homebrew/bin/npx"` 后仍会翻车：npx 本身是 `#!/usr/bin/env node` 的脚本，**启动时要再靠 PATH 解析 node**。GUI 应用从 Dock/开机自启动启动时 PATH 精简（launchd 默认 `/usr/bin:/bin:...`，不含 /opt/homebrew/bin），npx 找不到 node 当场退出——客户端报 `MCP error -32000: Connection closed`。极具迷惑性：**终端里原样跑命令是通的**（终端 PATH 齐全），会误判成服务器/网络/域名问题。

**修（终态形态，零 PATH 依赖 + 零启动期联网）**：安装时把 mcp-remote 固定下载到本机（`node npm-cli.js install --prefix ~/.trade-data-mcp mcp-remote`），配置直接用 node 二进制绝对路径 + 入口 js 绝对路径：
```json
{"command": "/opt/homebrew/bin/node",
 "args": ["/Users/me/.trade-data-mcp/node_modules/mcp-remote/dist/proxy.js",
          "http://1.2.3.4:8100/mcp", "--header", "Authorization: Bearer <token>", "--allow-http"]}
```
node 是二进制不是脚本，不再有第二跳 PATH 解析；`npx -y` 的"每次启动检查/下载"也一并消除（快、且离线可启动）。**验证要用 `env -i HOME=$HOME` 模拟 GUI 的精简环境跑桥**——终端直跑验证不出这个坑。

## 坑 2：缺 `--allow-http` → mcp-remote 拒连非 HTTPS 端点

`mcp-remote` **默认只允许 HTTPS 或 localhost**，对 `http://` 的非 localhost 地址直接拒绝：
```
Error: Non-HTTPS URLs are only allowed for localhost or when --allow-http flag is provided
```
测试环境端点常是 `http://`（没上 TLS），于是被静默挡在桥接层。

**修**：args 末尾加 `--allow-http`（端点已有 token 鉴权时可接受；生产建议上 HTTPS）：
```json
"args": ["-y", "mcp-remote", "http://1.2.3.4:8100/mcp",
         "--header", "Authorization: Bearer <token>", "--allow-http"]
```

## 正确配置（Claude Desktop / WorkBuddy schema 相同）

```json
"trade-data": {
  "command": "/opt/homebrew/bin/npx",
  "args": ["-y", "mcp-remote", "http://1.2.3.4:8100/mcp",
           "--header", "Authorization: Bearer <token>", "--allow-http"]
}
```
另需 Node ≥18（mcp-remote 要求）。config 路径：Claude Desktop=`~/Library/Application Support/Claude/claude_desktop_config.json`、WorkBuddy=`~/.workbuddy/mcp.json`（两者同 schema）；改完**重启客户端**。

## 坑 3：服务器重启后桥变"僵尸"——状态显示 running 但 "no tools available"

配置全对、曾经能用，某天客户端里 connector 显示 **"This connector has no tools available"**，而 Developer 面板里 server 状态却是 **running**。

**根因链**：mcp-remote 桥与远端保持 SSE 长连接 → 服务器重启/重建容器（`docker rm -f` 等）掐断连接 → mcp-remote 自动重连**只试 2 次**（`Maximum reconnection attempts (2) exceeded`，不可配）→ 放弃后**进程不退出**——本地 stdio 端还活着，GUI 就一直显示 running，但远端会话已死，工具列表拿不到/为空。

日志实锤（`~/Library/Logs/Claude/mcp-server-<name>.log`）：

```
Error: SSE stream disconnected: TypeError: terminated
  [cause]: SocketError: other side closed
Error: Maximum reconnection attempts (2) exceeded.
```

**修**：重启 GUI 客户端（Cmd+Q 重开），桥进程重建即恢复。**每次重新部署远端 MCP server 后，开着的 GUI 客户端都要重启一次**——把这步写进部署 runbook。"running" 只表示本地桥进程活着，不代表远端会话健康。

**定位口诀**：先在终端原样跑桥命令喂一条 `tools/list`（见下节）——终端里通、GUI 里空 → 必是客户端侧陈旧会话，直接重启客户端，别去改配置。

## 坑 4：僵尸桥会堆积——GUI 的"重连"按钮只生新不杀旧，堆多了连新桥都起不来

(2026-07-09 实测) 坑 3 的恶化形态：远端 server 重建后，用户在 GUI 里反复点"重连"，客户端**每次都 spawn 一组新的 npx+node 进程，旧的不回收**——一下午积了 12 个僵尸 mcp-remote（`ps aux | grep mcp-remote` 一屏都是）。此时新桥启动可能撞上残留进程/本地状态直接退出，GUI 报 `MCP error -32000: Connection closed`——症状升级了（坑 3 是"running 但没工具"，这里是干脆连不上），用户会误判成网络/IP/域名问题。

**修**：先杀光再重启客户端——
```bash
pkill -f "mcp-remote http://<你的host>"   # 清掉所有僵尸桥
# 然后 Cmd+Q 彻底退出 GUI 客户端，重新打开
```
**验**：修完在终端原样跑一遍桥命令（见下节），看到 `Proxy established successfully` 再让用户重启 GUI——先证明链路通，再动客户端，避免又一轮盲试。

**注意**：多个 GUI 客户端（Claude Desktop + WorkBuddy）共用同一远端时，pkill 会把所有客户端的桥都清掉——每个开着的客户端都要重启一次。

## 怎么快速定位（别靠重启客户端猜）

**直接在终端把客户端要跑的那条命令原样跑一遍**，看它自己的报错——比在 GUI 里反复"改配置→重启→看工具列表"快一个数量级：
```bash
/opt/homebrew/bin/npx -y mcp-remote http://1.2.3.4:8100/mcp \
  --header "Authorization: Bearer <token>" --allow-http
# 成功会打印: Connected to remote server / Proxy established successfully
# 失败直接吐 ENOENT / Non-HTTPS URLs... 一眼定位
```
GUI 客户端把子进程 stderr 藏起来了；把命令拎到终端跑，坑立刻现形。这是"GUI 里配置外部工具"类问题的通用调试法。

## 相关

- 端点侧鉴权见 [decision-records](../patterns/decision-records.md) 思路：公开的 http MCP 必须加 token（本例 Bearer），否则等于把后端数据裸奔给公网。
- 与 [ssh-config-as-global-host-alias](../patterns/ssh-config-as-global-host-alias.md) 同源：把"能跑的命令"沉淀成可复用配置，但要注意 GUI app 不继承你的 shell 环境。
