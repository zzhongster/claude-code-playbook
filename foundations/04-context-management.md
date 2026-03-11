# 上下文管理

## 核心原则

> Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills. — Anthropic 官方

上下文窗口是 Claude Code 最重要的资源。每条消息、每个读取的文件、每个命令输出都在消耗它。窗口快满时，Claude 会"遗忘"早期指令、犯更多错误。

## 关键操作

### `/clear` — 最常用的命令

不相关的任务之间，**必须 `/clear`**。这是最简单也最有效的上下文管理手段。

```
任务A: 修复登录 bug
/clear
任务B: 添加新 API 端点
/clear
任务C: 重构数据库层
```

### `/compact` — 压缩但保留关键信息

当会话很长但你还没完成任务时：

```
/compact 保留 API 变更的所有细节和测试命令
```

也可以在 CLAUDE.md 中配置压缩行为：

```markdown
当压缩上下文时，始终保留完整的已修改文件列表和测试命令。
```

### `/rewind` — 回到检查点

`Esc + Esc` 或 `/rewind` 打开回退菜单：
- 只恢复对话
- 只恢复代码
- 两者都恢复
- 从选定消息开始摘要

### `/btw` — 不污染上下文的快速提问

```
/btw Python 的 walrus operator 怎么写来着？
```

答案显示在浮层中，不进入对话历史。适合查一个快速事实。

## 子 Agent：上下文隔离的利器

> Since context is your fundamental constraint, subagents are one of the most powerful tools available. — Anthropic 官方

当 Claude 需要调研代码库时，它会读大量文件，全部消耗主对话的上下文。用子 Agent 把调研隔离：

```
用子 Agent 调查我们的认证系统怎么处理 token 刷新，
以及有没有现成的 OAuth 工具可以复用。
```

子 Agent 在独立上下文中探索，只返回摘要，不污染主对话。

实现验证也可以用子 Agent：

```
用子 Agent review 这段代码的边缘情况。
```

## 常见反模式

### 厨房水槽会话

一个会话里做了不相关的三件事，上下文全是无关信息。

**解决**：任务之间 `/clear`。

### 反复修正

Claude 做错了，你修正，还是错，再修正。上下文被失败的尝试污染。

**解决**：两次修正后如果还不对，`/clear` 然后写一个更好的初始 prompt（把学到的东西融入）。干净的会话 + 好 prompt 几乎总是胜过长会话 + 累积修正。

### 无限探索

让 Claude "调查一下"但没限定范围，它读了几百个文件。

**解决**：限定调查范围，或者用子 Agent 隔离。

## 会话管理技巧

```bash
claude --continue    # 继续最近的会话
claude --resume      # 从最近的会话列表中选择
```

用 `/rename` 给会话起描述性名字（如 `"oauth-migration"`），方便后续查找。

## 来源

- [Anthropic 官方 Best Practices](https://code.claude.com/docs/en/best-practices)
