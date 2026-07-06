# 给 GUI 客户端接远程 http MCP 的两个静默坑（npx 路径 + --allow-http）

> 把一个远程 streamable-http MCP server 接进 Claude Desktop / 腾讯 WorkBuddy 这类 **GUI 客户端**时，配置"看起来完全正确"但客户端里工具就是不出现——两个非报错、非直觉的坑，都不在文档显眼处。

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
