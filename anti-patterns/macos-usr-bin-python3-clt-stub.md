# 反模式：给非技术用户分发的脚本依赖 macOS 自带 `/usr/bin/python3`——没装 Xcode 命令行工具的机器一调用就 xcode-select 弹窗且命令不执行

## 一句话结论

macOS 自带的 `/usr/bin/python3`（以及 `/usr/bin/git`、`gcc`、`clang`、`make` 等）**不是真正的可执行程序，而是 Command Line Developer Tools (CLT) 的占位 shim**。没装 CLT 的机器一调用它，系统就弹"要安装开发者工具"对话框、往 stderr 打 `xcode-select: note: No developer tools were found, requesting install.`，而**命令本身没有真正执行**（非零退出）。开发机/技术人员的 Mac 都装过 CLT 所以永远测不出这个坑；普通同事的全新 Mac 没装，脚本跑到用 python3 的那步就静默失败。

## 场景

- 给不懂技术的同事分发 Mac 一键安装脚本（`.command`），用 `/usr/bin/python3` 做 JSON/配置文件处理（"反正 macOS 自带 python3"）
- 脚本前半段用独立安装的运行时（如 node）都绿，一到 python3 那步失败
- 报错里夹一行 `xcode-select: note: No developer tools were found, requesting install.`——这是唯一的诊断线索

## 踩坑经历

trade-data MCP 插件安装包（2026-07）：Mac 安装脚本用 `/usr/bin/python3` 合并写 Claude Desktop / WorkBuddy 的 JSON 配置。同事的 Mac 运行：清理进程 ✅、找 node ✅、下载插件核心 ✅、连接自检 ✅（全是 node 步骤），然后蹦出 `xcode-select: note: No developer tools were found...` + "Claude Desktop 配置失败"，最后还连带误报"没找到任何 AI 软件"（写配置失败→成功标志没置位→走了兜底分支，其实 Claude Desktop 是装了的）。根因：这台机器没装 CLT，`/usr/bin/python3` 是 shim，一调用触发安装弹窗且不执行。开发机测通是因为都装了 CLT。

## 解法

1. **别依赖 `/usr/bin/python3`**：给非技术用户的脚本，凡是"系统自带"的 CLT shim（python3/git/gcc/clang/make/…）都要当"可能不可用"处理。
2. **用已确认可用的运行时做处理**：本例插件本就要装 node，JSON 合并/文本处理全改用 `node -e`（node 是独立安装的真程序，不经 CLT；且天然处理 JSON）。TOML/文本删段用 node 正则、校验用 node 结构自检，一并摆脱 python3。
3. **要么显式装 CLT**：若非用某工具不可，脚本先 `xcode-select -p >/dev/null 2>&1 || xcode-select --install` 并等待——但这会弹窗要用户手动点、体验差，对"不懂技术的同事"是下策。优先选方案 2。
4. **验证要在最裸的环境**：能跑 ≠ 能装。目标用户的全新 Mac 没有 CLT，是与开发机的关键环境差异，必须单独考虑（同族教训见 ps51-utf8-no-bom-gbk-mangles-chinese：跨平台分发"在我机器上能跑"是最弱证据）。

## 教训

- macOS 的"自带 python3/git/gcc"是**开发者工具的诱饵 shim**，不是"系统组件"——把它们当默认可用，等于假设每台机器都装过 Xcode CLT，对普通用户不成立。
- 诊断线索就一行 `xcode-select: note: No developer tools were found`：看到它 = 某个 CLT shim 被调用而 CLT 没装，顺着往上找是哪条命令。
- 分发给非技术用户的东西，依赖面要按"刚开箱的 Mac"算：只有系统真正预装的（bash/zsh、基础 coreutils）和你自己脚本装的（node）才可靠；CLT 工具链一律不可假设。
