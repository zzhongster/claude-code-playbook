# 并行工作流

## 核心原则

一个 Claude 够用时就用一个。当任务可以真正并行时，用多个会话提速。关键是**判断什么时候该并行、用哪种方式并行**。

## 三种并行方式

### 1. 子 Agent（Subagent）

在同一会话内启动独立上下文的子任务。

```
用子 Agent 调查认证系统的 token 刷新机制。
```

**适合**：
- 调研/探索（不污染主上下文）
- 代码 review（独立视角）
- 需要读很多文件的任务

**不适合**：
- 需要修改文件（子 Agent 的修改不会自动合并）
- 需要和主对话共享状态

### 2. 多会话并行

开多个 Claude Code 实例，各自处理不同任务。

**桌面应用**：每个会话有独立的 worktree
**命令行**：

```bash
# 会话1：实现 OAuth
claude --resume oauth-migration

# 会话2（另一个终端）：修复 Bug
claude --resume fix-memory-leak
```

**Writer/Reviewer 模式**：

| 会话 A（Writer） | 会话 B（Reviewer） |
|-----------------|-------------------|
| 实现 rate limiter | |
| | Review rate limiter 代码，检查边缘情况和竞态条件 |
| 根据 review 反馈修改 | |

用另一个会话 review 的好处：**干净上下文不会偏向自己写的代码**。

### 3. 批量执行（Fan-out）

用 `claude -p` 非交互模式批量处理。

```bash
# 批量迁移文件
for file in $(cat files.txt); do
  claude -p "把 $file 从 React 迁移到 Vue。返回 OK 或 FAIL。" \
    --allowedTools "Edit,Bash(git commit *)"
done
```

**适合**：
- 大规模迁移（2000 个文件）
- 批量格式化/重构
- CI/CD 集成

**流程**：
1. 让 Claude 列出所有需要处理的文件
2. 写脚本循环调用 `claude -p`
3. 先在 2-3 个文件上测试，调整 prompt
4. 全量执行

## 选择指南

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 调研代码、读大量文件 | 子 Agent | 不污染主上下文 |
| 独立的两个功能开发 | 多会话 | 互不干扰 |
| 写代码 + Review | 多会话（Writer/Reviewer） | 独立视角 |
| 批量处理 N 个相似任务 | Fan-out `claude -p` | 适合自动化 |
| 需要全局意识的搜索/判断 | **不要并行** | 见 [单对话 vs 多 Worker](../patterns/single-conv-vs-multi-worker.md) |

## 注意事项

- 子 Agent 无法写入主 Agent 的文件 → 主 Agent 需要汇总
- 多会话操作同一文件可能冲突 → 用 worktree 隔离或分配不同文件
- `--dangerously-skip-permissions` 跳过所有权限检查 → 只在沙箱中使用
- 并行不等于更好 → 需要全局意识的任务不要拆（见 [碎片化 Worker 反模式](../anti-patterns/fragmented-workers.md)）

## Anthropic 团队的并行实践

- **产品设计团队**：自主循环 — Claude 写代码→跑测试→迭代，人在旁边偶尔指导
- **数据团队**：多实例跑不同仓库的不同项目，各自保持上下文，切换时无缝接续
- **推理团队**：Claude 生成测试→跑测试→修复→再测，直到全部通过

## 来源

- [Anthropic 官方 Best Practices](https://code.claude.com/docs/en/best-practices)
- [How Anthropic Teams Use Claude Code](https://claude.com/blog/how-anthropic-teams-use-claude-code)
