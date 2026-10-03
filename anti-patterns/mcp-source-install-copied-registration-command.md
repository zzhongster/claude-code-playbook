# 从源码装 MCP server 后照抄 `claude mcp add` 注册：四个静默坑

## 一句话结论

README 里那条 `claude mcp add <name> -- node /path/index.js` 在终端里跑 `claude mcp list` 会显示 ✔ Connected，但它**默认只在当前目录生效**，**裸 `node` 在桌面 App 的精简 PATH 下起不来**；而 `npm install` 会**先跑完 install/prepare 脚本你才来得及审代码**，`git pull` 还会被 npm 自动改动的 lock 文件挡住。正确做法：先审源码再 install、用 stdio 握手 + `env -i` 冒烟、`-s user` + 绝对路径注册。

## 场景

2026-10-03，在 macOS 26.4 / Apple 芯片上给 Claude Code（桌面 App Code tab）装 Compositor 的 MCP server（`github.com/Josusanz/compositor-mcp`，TypeScript，依赖 `@modelcontextprotocol/sdk` + `sharp` + `zod`）。npm 上有多个同名 `compositor-mcp` 包，所以不用 `npx -y`，改为 clone 源码后 `npm install && npm run build` 再注册。

用户给的注册命令是 `claude mcp add compositor -- node /Users/<me>/compositor-mcp/dist/src/index.js`。这台机器的 node 不是 Homebrew 装的，而是 `~/.local/bin/node`，是一个单独放的二进制。

> 四个坑里，**坑 1 有实测复现**；坑 2–4 是在执行前识别出来的（坑 4 的 lock 改动已实际发生，只是还没触发 pull 冲突）。

## 坑 1：裸 `node` → 终端绿灯，GUI 里起不来（已实测）

`claude mcp list` 的健康检查是在**你当前 shell 的 PATH** 下启动 server 的，所以显示 ✔ Connected。桌面 App 从 Dock 启动时，PATH 是 launchd 的精简版（`/usr/bin:/bin:/usr/sbin:/sbin`），不包含 `~/.local/bin`、`/opt/homebrew/bin`、nvm 等目录。

用精简环境模拟复现：

```bash
env -i HOME=$HOME PATH=/usr/bin:/bin:/usr/sbin:/sbin node ~/compositor-mcp/dist/src/index.js
# → env: node: No such file or directory

env -i HOME=$HOME PATH=/usr/bin:/bin:/usr/sbin:/sbin ~/.local/bin/node ~/compositor-mcp/dist/src/index.js
# → 正常握手，tools/list 返回 15 个工具
```

**修**：注册时 node 写绝对路径（用 `which node` 查）。原理与 [GUI 客户端接远程 http MCP 的坑](mcp-remote-http-client-gotchas.md) 相同。**`claude mcp list` 的绿灯不能证明 GUI 里能用**，要用 `env -i` 验证。

## 坑 2：`claude mcp add` 默认是 local 作用域

不带 `-s` 时，配置写进 `~/.claude.json` 里**当前工作目录对应的项目条目**下，只在这个目录的会话里可用。换个项目目录开会话，工具就不出现，也不报错。

**修**：想全局可用就加 `-s user`。注册完用 `claude mcp get <name>` 检查，应显示 `Scope: User config (available in all your projects)`。

## 坑 3：`npm install` 会先执行代码，审查要在它之前

`npm install` 会运行：①依赖包的 `preinstall/install/postinstall`；②本包 `package.json` 里的 `prepare`（本例是 `tsc`，install 时就编译了一次）。也就是说，「装完再看看代码有没有问题」的顺序是反的。

**做法：clone 之后、install 之前**，只做只读审查：

```bash
# 1. 源码里的网络 / 子进程 / 文件系统 / 动态执行
grep -nE "fetch|https?:|net\.|child_process|exec|spawn|process\.env|homedir|rm\(|unlink|writeFile|eval|Function\(" src/*.ts

# 2. lock 文件的下载源是否全部来自官方 registry
grep -o '"resolved": "https\?://[^/]*' package-lock.json | sort | uniq -c

# 3. 哪些依赖会在 install 时跑脚本
python3 -c "import json;d=json.load(open('package-lock.json'))
[print(k,v.get('version')) for k,v in d['packages'].items() if v.get('hasInstallScript')]"

# 4. package.json 的 scripts 里有没有 preinstall/postinstall/prepare
```

本例结果：没有任何网络请求；唯一的子进程调用是 `execFile("open", ["-b", bundleId, path])`，参数按数组传、不经过 shell；删除操作只限 `.comp/images/` 内；178 个包全部来自 `registry.npmjs.org`；带 install 脚本的只有 `esbuild` 和 `fsevents`（都是 devDependencies 带进来的常见包）。

## 坑 4：npm 改了 lock 文件 → 下次 `git pull` 被挡住

原作者发版时没有同步 lock 里的版本号（`package.json` 是 0.1.4，lock 里还是 0.1.0），`npm install` 会自动修正，于是工作区多出一处 `M package-lock.json`。以后 `git pull` 时，如果上游也改了 lock，pull 会直接拒绝。

**修**：更新流程固定为：

```bash
cd ~/compositor-mcp && git checkout package-lock.json && git pull && npm install && npm run build
```

如果要长期改代码，就 fork + 开分支，把原仓库设为 `upstream`，用 `git fetch upstream && git merge upstream/main` 同步。

## 注册前的冒烟测试（不依赖客户端）

stdio MCP server 可以直接用管道喂 JSON-RPC 测试，不用先注册、重启会话再去看工具有没有出现：

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"t","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
| env -i HOME=$HOME PATH=/usr/bin:/bin:/usr/sbin:/sbin /abs/path/node dist/src/index.js
```

能拿到 `id:2` 的 tools 列表，就说明「依赖能加载（比如 sharp 的原生模块）+ GUI 环境下能启动」两件事都成立。stdin 关闭后 server 会自行退出，属于正常现象。

## 适用范围

- 适用于：任何从源码安装、以 stdio 方式注册到 Claude Code / Claude Desktop / 其他 GUI 客户端的 MCP server（node / python / uv 都适用）。
- 不适用于：远程 http MCP（见 [mcp-remote-http-client-gotchas](mcp-remote-http-client-gotchas.md)）；以及只在终端 CLI 里用、PATH 完整的场景（坑 1 不会触发，但坑 2–4 仍然适用）。

## 相关

- [GUI 客户端接远程 http MCP 的四个静默坑](mcp-remote-http-client-gotchas.md)：GUI 精简 PATH 的同源问题
- [macOS /usr/bin/python3 是 CLT 存根](macos-usr-bin-python3-clt-stub.md)：精简 PATH 下解释器解析到错误位置的同类问题
