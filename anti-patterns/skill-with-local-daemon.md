# 反模式：装带本地 daemon 的 Skill 不配套关闭手段

## 一句话结论

装一个会启后台 HTTP 服务的 skill（比如 web-access 的 CDP Proxy），如果不配套写"一键关闭脚本"+ 留意它依赖的系统级开关，就等于在你的机器上长期开了一个零认证的本地控制后门——而你自己都不知道它还开着。

## 场景

社区流行的 skill 越来越多，里面有一类是"给 Claude 真正的浏览器/系统能力"的，典型代表 [web-access](https://github.com/eze-is/web-access)：

- 跑一个常驻后台进程（detached + unref，Claude Code 退出它不退出）
- 监听本地端口提供 HTTP API（如 `localhost:3456`）
- API 包含 `/eval`（任意 JS）、`/setFiles`（读本机任意路径）等高权限端点
- 要求你打开浏览器层的某个系统开关（如 Chrome 的 `Allow remote debugging`）才能用

装的时候只关心"它能干什么"，不去想"用完怎么关、关不干净有什么后果"。

## 踩了什么坑

实际检查一个装了几周、自己都忘了什么时候装的 skill，发现：

1. **本地 HTTP 服务零鉴权**：监听 127.0.0.1 看似安全，但任何本机进程（其它 CLI、npm 包、甚至打开的网页通过 `fetch('http://localhost:3456/eval')`）都能调用，等于把"在用户登录态下执行任意 JS"开放给所有本机进程
2. **持久化后台进程**：用 `spawn(..., { detached: true })` + `child.unref()` 启动，Claude 退了它继续跑，没有自动停止机制，文档还写"不建议主动停止"
3. **依赖的系统级开关被遗忘开着**：Chrome `Allow remote debugging` 一旦勾上，任何能连 9222 的本地进程都能完全控制浏览器（读所有 cookie / 标签内容），这个风险面**和 skill 是否在跑无关**
4. **没有 git 历史**：手动安装的 skill 目录通常没有 .git，无法验证装的是不是被篡改的版本，也没法 diff 升级前后的变化

## 根因

skill 生态目前还在"展示能力"阶段，README 都在讲"我能干什么"，没人讲"用完怎么收拾"。安装文档负责让你能跑起来，停止流程没人写。

而带 daemon 的 skill 和"无副作用的工具调用"完全不是一类东西：

| | 普通 skill | 带 daemon 的 skill |
|---|---|---|
| 生命周期 | 跟着对话 | 独立于 Claude Code 进程 |
| 攻击面 | 调用时才存在 | 持续暴露 |
| 依赖的系统配置 | 通常无 | 常需要打开系统级开关 |
| 用户心智 | 装了就装了 | **必须有"关"的概念** |

## 替代方案

装这类 skill 时同时做三件事：

### 1. 第一时间写"一键关闭"脚本

至少做这两件事：

```bash
#!/usr/bin/env bash
# 1. kill 后台进程（先 SIGTERM，1 秒不死再 SIGKILL）
pids=$(pgrep -f 'cdp-proxy\.mjs' || true)
[ -n "$pids" ] && kill $pids && sleep 1
remaining=$(pgrep -f 'cdp-proxy\.mjs' || true)
[ -n "$remaining" ] && kill -9 $remaining

# 2. 检查它依赖的系统开关是否还开着，开着就提示并打开设置页
for port in 9222 9229 9333; do
  if lsof -nP -iTCP:"$port" -sTCP:LISTEN >/dev/null 2>&1; then
    echo "⚠️  Chrome 调试端口 $port 仍开放，去 chrome://inspect 取消勾选"
    open 'chrome://inspect/#remote-debugging'
  fi
done
```

加 alias 让随手关变成肌肉记忆：

```bash
alias wastop='~/.agents/skills/web-access/stop.sh'
```

### 2. 给 skill 目录建 git 基线

skill 装好第一时间：

```bash
cd ~/.agents/skills/<skill-name>
git init && git add -A && git commit -m "baseline v<version>"
```

这样后续：
- `git status` 能看到文件被外部修改
- 升级时 `git diff` 能看到改了什么
- 自己改了配置（比如想加 token 鉴权）能用 git 管理，避免被升级覆盖

### 3. 区分"日常开"和"用完关"两类系统开关

像 Chrome 的 `Allow remote debugging` 这种**改变浏览器全局安全模型**的开关，永远默认关闭，用之前再开。不要为了"省事"长期开着——99% 的时间你不用 web-access，但风险面 100% 的时间在那。

## 适用范围

- **适用于**：所有会启后台进程 / 监听端口 / 要求你打开系统级安全开关的 skill 或 MCP server
- **不适用于**：纯函数式的 skill（只在调用时执行，无状态、无后台进程、无系统配置变更）

## 相关

- 上游 skill：<https://github.com/eze-is/web-access>
- 不要把"加 token 鉴权"当成万能解药：能堵网页 CSRF 和多用户系统下的别人，但堵不住以你身份运行的恶意进程（它能读到 token 文件）。**关进程 + 关系统开关**才是根治
