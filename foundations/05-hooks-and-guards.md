# Hooks 做确定性保障

## 核心原则

> CLAUDE.md is advisory; Hooks are deterministic. — Anthropic 官方

CLAUDE.md 里的规则是"建议"，Claude 可能遵守也可能忽略（特别是文件太长时）。Hooks 是在 Claude 工作流特定节点**自动运行的脚本**，100% 确定性执行。

## CLAUDE.md vs Hooks

| | CLAUDE.md | Hooks |
|--|-----------|-------|
| 性质 | 建议性 | 确定性 |
| 执行 | Claude 自行判断是否遵守 | 系统自动运行，不经 Claude |
| 适合 | 代码风格偏好、架构模式 | 必须每次执行的检查、自动化 |
| 失败时 | Claude 可能忽略 | 阻断工作流 |

## Hook 类型

| 触发点 | 用途 |
|--------|------|
| 文件编辑后 | 跑 linter、格式化、类型检查 |
| 命令执行前 | 拦截危险命令（如 `rm -rf`） |
| 提交前 | 跑测试、检查敏感信息 |
| 工具调用后 | 日志记录、监控 |

## 实践示例

### 编辑后自动 lint

让 Claude 帮你写：

```
写一个 hook，每次文件编辑后自动跑 eslint。
```

或者手动配置 `.claude/settings.json`：

```json
{
  "hooks": {
    "afterEdit": [
      {
        "command": "npx eslint --fix ${file}",
        "description": "Auto-lint after edit"
      }
    ]
  }
}
```

### 阻止修改敏感目录

```
写一个 hook，阻止写入 migrations/ 目录下的文件。
```

### 提交前跑测试

```
写一个 hook，在 git commit 前自动跑 npm test。
```

## 配置方式

- `/hooks` — 交互式配置
- 直接编辑 `.claude/settings.json`
- 让 Claude 帮你写：`"Write a hook that..."`

## 何时用 Hook vs CLAUDE.md

| 需求 | 用什么 |
|------|--------|
| "改完代码后跑 typecheck" | **Hook**（必须执行） |
| "用 camelCase 命名变量" | **CLAUDE.md**（风格建议） |
| "不要修改 migrations 目录" | **Hook**（硬性禁止） |
| "优先使用函数式组件" | **CLAUDE.md**（模式偏好） |
| "commit 前跑 lint" | **Hook**（必须执行） |
| "commit message 用中文" | **CLAUDE.md**（格式建议） |

## 来源

- [Anthropic 官方 Hooks Guide](https://docs.anthropic.com/en/docs/claude-code/hooks-guide)
- [Anthropic 官方 Best Practices](https://code.claude.com/docs/en/best-practices)
