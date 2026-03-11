# CLAUDE.md 写法

## 核心原则

CLAUDE.md 是 Claude Code 每次会话开始时自动读取的文件。它的作用是给 Claude **它无法从代码中推断出来的上下文**。

### 精简至上：200 行以内

> CLAUDE.md should target under 200 lines per file — shanraisshan/claude-code-best-practice

> Bloated CLAUDE.md files cause Claude to ignore your actual instructions! — Anthropic 官方文档

对每一行问自己：**"删掉这行 Claude 会犯错吗？"** 如果不会，删掉。

### 该写什么 vs 不该写什么

| ✅ 该写 | ❌ 不该写 |
|---------|----------|
| Claude 猜不到的 Bash 命令（构建、测试、部署） | Claude 读代码就能知道的东西 |
| 和默认不同的代码风格规则 | 语言的标准约定 |
| 测试框架和偏好的 test runner | 详细的 API 文档（给链接就好） |
| 分支命名、PR 规范、commit 格式 | 经常变化的信息 |
| 项目特有的架构决策 | 长篇解释和教程 |
| 环境变量、开发环境的坑 | 逐文件的代码描述 |
| 常见陷阱和非直觉行为 | "写干净代码"之类的废话 |

### 示例

```markdown
# 构建和测试
- `npm run build` 构建项目
- `npm test -- --watch` 运行测试（推荐 watch 模式）
- `npm run lint` 检查代码风格

# 代码风格
- 使用 ES modules (import/export)，不用 CommonJS (require)
- 解构导入：`import { foo } from 'bar'`
- 变量命名：camelCase（函数/变量）、PascalCase（类/组件）

# 工作流
- 代码改完后跑 typecheck
- 优先跑单个测试文件，不要跑全套
- commit message 用中文，格式：`feat: 功能描述`

# 架构决策
- API 路由在 src/routes/，中间件在 src/middleware/
- 数据库用 Prisma ORM，迁移文件不要手动改
- 环境变量在 .env.example 有完整列表
```

## 进阶技巧

### 多文件策略

```
项目根/CLAUDE.md          ← 全局规则（所有人共享）
项目根/packages/api/CLAUDE.md  ← API 子项目的规则
~/.claude/CLAUDE.md        ← 个人偏好（不提交到 git）
```

子目录的 CLAUDE.md 会在 Claude 处理该目录文件时自动加载。Monorepo 用这个分割上下文很有效。

### 用 @ 引用其他文件

```markdown
项目概览见 @README.md
NPM 命令见 @package.json
Git 工作流见 @docs/git-instructions.md
```

### 强调关键规则

对必须遵守的规则加强调：

```markdown
IMPORTANT: 永远不要直接修改 migrations/ 目录下的文件
YOU MUST: 每次修改 API 后运行 `npm run generate-types`
```

### 版本控制

**把 CLAUDE.md 提交到 git**。整个团队受益，随项目演进。当 Claude 反复犯同一类错误时，就在 CLAUDE.md 里加一条规则。

### 定期修剪

CLAUDE.md 会随时间膨胀。每月审查一次：
- Claude 已经能自动做对的规则 → 删除
- 过时的命令或路径 → 更新
- 太长的解释 → 精简成一行

## 来源

- [Anthropic 官方 Best Practices](https://code.claude.com/docs/en/best-practices)
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
- [HumanLayer: Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Trail of Bits: claude-code-config](https://github.com/trailofbits/claude-code-config)
