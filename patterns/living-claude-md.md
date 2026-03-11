# Living CLAUDE.md：与代码同步的活文档

## 核心原则

> CLAUDE.md 不是写一次就完的 README，它是代码的"运行手册"，必须跟代码保持同步。

CLAUDE.md 的价值取决于它的**准确性**。一旦它和代码脱节，Claude 会按过时的指令工作，产生更多错误。

## 铁律：改代码 = 改 CLAUDE.md

来自一个量化交易系统的实践（10+ 模块，30+ 参数）：

```
任何修改以下模块的代码，必须同步更新 CLAUDE.md 中对应的参数值，不可跳过。
- 涉及文件：presets.py, scorer.py, fundamental.py, technical.py, ...
- 对应章节映射见 CLAUDE.md §九
```

为什么需要这么严格？因为这个系统中：
- 评分公式 `S_total = w_F * F_Score + w_T * T_Score + w_M * M_Score` 的权重在代码里
- CLAUDE.md 里记录了这些权重的当前值
- 如果代码改了权重但 CLAUDE.md 没更新，下次 Claude 读到旧值就会产出错误的分析

## 实践方法

### 1. 建立章节-文件映射表

在 CLAUDE.md 中明确哪些章节对应哪些代码文件：

```markdown
## 参数映射
| CLAUDE.md 章节 | 对应代码文件 | 关键参数 |
|---------------|------------|---------|
| §十 评分体系 | scorer.py | w_F, w_T, w_M |
| §十一 状态机 | state_machine.py | 阈值, 转换条件 |
| §十二 风控 | engine.py | 止损比例, 仓位上限 |
```

这样 Claude 在修改代码时，能自动知道 CLAUDE.md 哪个章节需要同步更新。

### 2. 在 CLAUDE.md 中声明同步规则

```markdown
# 铁律
改代码时必须同步更新本文件对应章节。这不是建议，是硬性要求。
```

放在 CLAUDE.md 最前面，确保 Claude 每次都能看到。

### 3. 用 Hook 辅助检查（可选）

如果想要确定性保障，可以用 Hook 在提交前检查 CLAUDE.md 是否被修改：

```json
{
  "hooks": {
    "beforeCommit": [{
      "command": "git diff --cached --name-only | grep -q 'CLAUDE.md' || echo 'WARNING: 代码改了但 CLAUDE.md 没更新'",
      "description": "Check CLAUDE.md sync"
    }]
  }
}
```

## 什么内容需要同步

| 内容类型 | 示例 | 同步必要性 |
|---------|------|-----------|
| 参数值/阈值 | 评分权重、超时时间、批次大小 | **必须** |
| 架构决策 | 新增/删除模块、改变数据流 | **必须** |
| 命令/脚本 | 启动命令、测试命令 | **必须** |
| 端口/URL | 服务端口、API 路径 | **必须** |
| 代码风格偏好 | 命名规范、模式偏好 | 变动时更新 |
| 背景知识 | 业务概念解释 | 很少变 |

## 反模式

### "写完就忘"型 CLAUDE.md

项目初期写了一份详细的 CLAUDE.md，后续代码迭代了 20 次但 CLAUDE.md 一次没动。Claude 按照初版指令工作，产出和当前代码矛盾。

### "百科全书"型 CLAUDE.md

把所有代码细节都写进 CLAUDE.md（300+ 行），导致关键信息被淹没，更新成本极高。

**解决**：CLAUDE.md 只放"Claude 做决策时需要的信息"（见 [CLAUDE.md 写法](../foundations/01-claude-md.md)），细节留在代码注释里。

## 配合讨论记录

同一个项目还有一条铁律：

> 每次对话中涉及的讨论、决策、分析结论，必须记入 discussions.md，便于后续复盘。

CLAUDE.md 管"当前状态"，discussions.md 管"为什么变成这样"。两者配合形成完整的项目记忆。

## 来源

- SAWSystem 量化交易系统开发实践（2026-02 ~ 2026-03）
- [CLAUDE.md 写法最佳实践](../foundations/01-claude-md.md)
