# 给 Claude 验证手段

## 核心原则

> Giving Claude verification criteria is the single highest-leverage practice. — Anthropic 官方

Claude 能自我验证时，表现会好几个数量级。没有验证手段，它只能产出"看起来对"的代码，你变成唯一的反馈循环。

## 验证手段层级

| 验证方式 | 自动化程度 | 效果 |
|---------|-----------|------|
| 测试套件（unit/integration） | 全自动 | ★★★★★ |
| Linter / TypeCheck | 全自动 | ★★★★ |
| 截图对比（UI） | 半自动 | ★★★★ |
| 构建成功 | 全自动 | ★★★ |
| 人工 review | 手动 | ★★ |

## 实践方式

### 在 prompt 中给验证标准

```diff
- "实现一个验证邮箱的函数"

+ "写一个 validateEmail 函数。
+  测试用例：user@example.com → true, invalid → false, user@.com → false
+  实现后跑测试。"
```

### 让 Claude 写测试再写代码

```
先写测试，确保测试能跑但失败。
然后写实现代码让测试通过。
最后跑全套测试确保没有回归。
```

这是 Anthropic 安全团队自己用的流程：从"先写代码再补测试"转变为"先写测试再写代码"。

### UI 变更用截图验证

```
[粘贴设计稿截图]

实现这个设计。完成后截图对比原图，列出差异并修复。
```

配合 [Claude in Chrome 扩展](https://code.claude.com/docs/en/chrome)，Claude 可以自动打开浏览器、截图、对比。

### 构建 + Lint 作为底线

在 CLAUDE.md 中：

```markdown
# 工作流
- 改完代码后跑 `npm run typecheck`
- 提交前跑 `npm run lint`
```

或者用 Hook 强制执行（见 [Hooks](05-hooks-and-guards.md)）。

## Anthropic 团队的验证实践

- **安全团队**：测试驱动开发，先写测试再写代码
- **产品团队**：自主循环 — Claude 写代码→跑测试→迭代，直到所有测试通过
- **推理团队**：Claude 生成的单元测试覆盖边缘情况，减少 80% 研发周期

## 反模式

- **信任但不验证**：Claude 产出看起来合理的代码，但不处理边缘情况 → 总是提供验证手段
- **只靠人工 review**：你的注意力是稀缺资源 → 尽量用自动化验证

## 来源

- [Anthropic 官方 Best Practices](https://code.claude.com/docs/en/best-practices)
- [How Anthropic Teams Use Claude Code](https://claude.com/blog/how-anthropic-teams-use-claude-code)
