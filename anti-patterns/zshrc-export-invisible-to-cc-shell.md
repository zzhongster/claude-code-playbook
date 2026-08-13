# 把环境变量写 ~/.zshrc，Claude Code 的 shell 根本读不到

> 来源：TOUCH 2.0 /notify-group 首测实踩（2026-08-13，TOU-345）

## 现象

给发群命令配 webhook：往 `~/.zshrc` 追加 `export WECOM_WEBHOOK_URL=...`，下一个 Bash 工具调用里该变量为空，命令按设计停机。

## 根因

`~/.zshrc` 只被**交互式** zsh 读取；Claude Code 的 Bash 工具是非交互 shell，不加载它。终端里手敲一切正常，CC 会话里就是拿不到——两个世界。

## 正确做法

Claude Code 会话要用的环境变量，配在**用户主目录**的 `~/.claude/settings.json`：

```json
{ "env": { "KEY": "value" } }
```

CC 主动注入每个新会话，全平台一致（Windows 是 `C:\Users\<你>\.claude\settings.json`）。两个附带的坑：

- 认准用户主目录那份——仓库里往往也有 `.claude/settings.json`，写错地方密钥会被 commit；
- 配完要**重开会话**才生效（注入发生在会话启动时）。

## 更一般的教训

**写配置类文档时，每条路径先跑通再写。** 这次是把「终端里通用的做法」直接抄成「对 CC 也有效」，没实测——文档发出去，照做的每个人都会在同一处失败。属于「回读实际值，不信任何自述」的文档变体：你写下的操作步骤也是一种自述。
