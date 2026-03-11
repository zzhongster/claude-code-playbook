# Claude Code Playbook

用 Claude Code 编程的工程实践手册。不是功能列表，不是 tips 汇总，而是**经过实战验证、带数据、可复用的工程决策**。

## 为什么做这个

市面上的 Claude Code 资源多是功能罗列型（命令大全、awesome 清单）。但真正影响效率的不是知道多少命令，而是**工程决策**——什么时候用子 Agent、什么时候单对话、怎么写 prompt 让 LLM 不偷懒。

这些决策需要实战验证，需要数据支撑，需要踩过坑才能总结出来。

## 内容结构

```
claude-code-playbook/
├── foundations/       ← 基础共识（官方 + 社区公认的实践）
├── patterns/          ← 工程模式（经过验证的做法）
├── anti-patterns/     ← 反模式（踩过的坑）
├── experiments/       ← 对比实验（带数据的验证）
└── templates/         ← 可复用模板
```

### Foundations — 基础共识

| 文档 | 主题 |
|------|------|
| [CLAUDE.md 写法](foundations/01-claude-md.md) | 项目入口文件的最佳实践 |
| [先规划再写码](foundations/02-plan-before-code.md) | Plan Mode 的正确用法 |
| [给验证手段](foundations/03-verification.md) | 测试、截图、lint — Claude 能自我验证才能自主工作 |
| [上下文管理](foundations/04-context-management.md) | 窗口有限，如何高效利用 |
| [Hooks 做确定性保障](foundations/05-hooks-and-guards.md) | CLAUDE.md 是建议，Hooks 是强制 |
| [并行工作流](foundations/06-parallel-workflows.md) | 子 Agent、worktree、batch 的选择 |

### Patterns — 工程模式

| 文档 | 核心结论 |
|------|---------|
| [Prompt > 模型 > 架构](patterns/prompt-over-model.md) | 同一模型优化 prompt 的效果 > 换更贵的模型 |
| [单对话 vs 多 Worker](patterns/single-conv-vs-multi-worker.md) | 需要全局意识的任务用单对话，独立任务才拆 |
| [四阶段搜索工作流](patterns/four-stage-search.md) | 广搜→追线索→覆盖检查→输出 |

### Anti-patterns — 反模式

| 文档 | 踩坑场景 |
|------|---------|
| [碎片化多 Worker](anti-patterns/fragmented-workers.md) | 拆太碎导致每个 Worker 都不够聪明 |
| [便宜模型做判断](anti-patterns/cheap-model-judgment.md) | 省钱模型在筛选/判断任务上翻车 |

### Experiments — 对比实验

| 文档 | 实验内容 |
|------|---------|
| [Skill vs Pipeline 全量对比](experiments/2026-03-12-skill-vs-pipeline.md) | 4 种架构方案的完整数据 |

## 内容来源

- **一手实验**：自己的 Claude Code 项目实战（带数据）
- **官方文档**：[Anthropic Best Practices](https://code.claude.com/docs/en/best-practices)、[How Anthropic Teams Use Claude Code](https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf)
- **社区精华**：从高星 repo 和文章中提炼共识

## 使用方式

1. **新项目启动前**：读 foundations/ 建立基础
2. **遇到架构决策时**：查 patterns/ 找参考
3. **踩坑后**：查 anti-patterns/ 看别人是不是也踩过
4. **想验证某个假设时**：参考 experiments/ 的实验方法

## 贡献

欢迎提交你的实战经验！详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## License

MIT
