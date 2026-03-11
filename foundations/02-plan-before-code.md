# 先规划再写码

## 核心原则

> The single most important principle is never letting Claude write code until you've reviewed and approved a written plan.

让 Claude 直接写代码，它可能解决的是错误的问题。**先 Plan Mode 探索，再 Normal Mode 实现**。

## 四步工作流

### 1. 探索（Plan Mode）

```
claude (Plan Mode)
> 读 /src/auth 目录，理解我们怎么处理 session 和登录。
> 也看看环境变量怎么管理 secrets。
```

让 Claude 读代码、回答问题，但不做任何修改。

### 2. 规划（Plan Mode）

```
claude (Plan Mode)
> 我想加 Google OAuth。哪些文件需要改？Session 流程是什么？做个计划。
```

按 `Ctrl+G` 在编辑器中打开计划，直接编辑后再让 Claude 执行。

### 3. 实现（Normal Mode）

```
claude (Normal Mode)
> 按你的计划实现 OAuth 流程。给 callback handler 写测试，
> 跑测试套件，修复所有失败。
```

### 4. 提交

```
claude (Normal Mode)
> 用描述性的 commit message 提交，开一个 PR。
```

## 什么时候可以跳过规划

规划有开销。以下场景可以直接做：

| 场景 | 直接做还是先规划 |
|------|----------------|
| 修个 typo、加行日志 | 直接做 |
| 改一个文件的简单重构 | 直接做 |
| 你能一句话描述完整 diff | 直接做 |
| 不确定怎么做 | **先规划** |
| 修改多个文件 | **先规划** |
| 不熟悉被改的代码 | **先规划** |
| 涉及架构变更 | **先规划** |

## 进阶：让 Claude 采访你

对于复杂功能，不要试图一次写完 prompt。让 Claude 来问你：

```
我想做 [简要描述]。用 AskUserQuestion 工具详细采访我。

问技术实现、UI/UX、边缘情况、担忧和取舍。
不要问显而易见的问题，深入我可能没考虑到的难点。

一直采访直到全部覆盖，然后把完整 spec 写到 SPEC.md。
```

采访完成后，**开一个新会话**执行 spec。新会话有干净的上下文，专注于实现。

## 跨模型审查

计划写完后，可以用另一个模型（如 Codex、另一个 Claude 实例）审查方案质量，避免单一模型的盲区。

## 来源

- [Anthropic 官方 Best Practices](https://code.claude.com/docs/en/best-practices)
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
