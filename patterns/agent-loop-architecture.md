# Agent Loop 架构模式 — 从 Claude Code 源码学到的

> 来源：Claude Code CLI 源码深度分析（2026-04-15）
> 文件：/Users/zhangzhong/Developer/claude-code/src/

## 核心发现

### 1. Generator 状态机驱动 Agent Loop

Claude Code 的核心循环不是简单的 while 循环，而是一个 **generator-based state machine**：

```typescript
// query.ts:268
let state: State = { messages, toolUseContext, turnCount, transition }
// 每轮迭代只修改 state 的一部分，其余保持不变
state = { ...state, ...updates }
```

**启发**：我们的 Deep Research Agent 应该用同样的模式——每轮研究只修改研究状态的一部分（已搜索来源、已获取数据、待填字段），其余保持不变。这让重试和恢复变得简单。

### 2. 多轮工具调用的退出条件

```
循环退出条件：
1. Token 预算用完（checkTokenBudget）
2. 模型决定不再调用工具（返回纯文本）
3. 用户中断
4. 连续错误超过阈值
5. 收益递减（连续 <500 token 新内容）
```

**启发**：Research Agent 的退出条件应该是：
- 所有必填字段都有值（质量评分 ≥ B）
- 搜索了 3 轮仍找不到 → 标"未查到"
- Token 预算用完
- 用户手动停止

### 3. 工具并发安全

```typescript
// StreamingToolExecutor
isConcurrencySafe(input): boolean
// 只有标记为并发安全的工具才会并行执行
```

**启发**：Research Agent 的工具中，`web_search` 和 `platform_search` 是并发安全的（可以同时搜多个关键词）；`save_report` 不是（要等所有数据就绪）。

### 4. System Prompt 的缓存边界

```typescript
// prompts.ts:114
SYSTEM_PROMPT_DYNAMIC_BOUNDARY
// 静态部分（工具描述、角色定义）缓存
// 动态部分（当前任务、已搜索结果）不缓存
```

**启发**：Research Agent 的 System Prompt 应该分两部分：
- **静态**（缓存）：研究方法论、工具描述、输出格式
- **动态**（不缓存）：当前公司名、调研方向、已获取的数据

### 5. Coordinator 的编排模式

```
coordinatorMode.ts 的核心指导：
1. 永远不要委托理解（"Never delegate understanding"）
2. Worker 是异步的，可以并行启动
3. 综合发现是 Coordinator 的责任，不是 Worker 的
4. Continue vs. Spawn 有明确的决策标准
```

**启发**：Research Agent 不应该让 LLM 做所有事情。它应该：
- 自己理解和综合数据（不委托给子 Agent）
- 工具调用可以并行（搜索多个来源）
- 最终报告的组装是 Agent 自己做的

### 6. 消息为中心的状态

```
一切通过 messages 数组流转：
- 工具结果 → tool_result message
- 压缩边界 → 标记在 messages 中
- 停止信号 → 通过 messages 传递
```

**启发**：Research Agent 的研究状态应该完全体现在 messages 中——每次工具调用的结果都作为 assistant/tool message 追加，LLM 可以看到完整的研究历史来决定下一步。

## 应用到 Deep Research Agent

```python
class DeepResearchAgent:
    """仿 Claude Code 架构的深度研究 Agent"""

    async def research(self, company, direction):
        # 1. 静态 prompt（缓存）+ 动态 prompt（不缓存）
        system = self.STATIC_PROMPT + f"\n当前任务：{company}，方向：{direction}"

        # 2. Generator 状态机
        state = ResearchState(company, direction)
        messages = [{"role": "user", "content": f"调研 {company}"}]

        # 3. 多轮循环，退出条件明确
        for round in range(MAX_ROUNDS):
            resp = await llm.chat(messages=messages, tools=self.tools)

            if resp.tool_calls:
                # 4. 并发安全的工具并行执行
                results = await self.execute_tools_concurrent(resp.tool_calls)
                messages.extend(results)
                state.update(results)
            else:
                # 模型认为研究完成
                break

            # 5. 收益递减检查
            if state.quality_score >= TARGET and state.no_new_data_rounds >= 2:
                break

        # 6. Agent 自己综合报告（不委托）
        return state.compile_report()
```
