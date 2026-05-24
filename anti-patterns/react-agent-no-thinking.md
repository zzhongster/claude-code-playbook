# ReAct Agent ≠ Deep Reasoning — 推理模式反而毁掉 Agent

## TL;DR

给 ReAct 风格的 agent loop（多轮 `THOUGHT/ACTION/OBSERVATION`）开思维链 / thinking 模式，
**结果质量下降 + 耗时翻 3-5 倍 + 格式遵守变差**。看起来"推理强"应该帮 agent，实测相反。

## 场景

跑一个 qwen3.6-plus 驱动的 milestones 调研 agent：8 轮预算，工具 `web_search` + `scrape_url`，
ReAct 格式（`THOUGHT: ... ACTION: web_search("...")`），找出公司发展节点。

直觉：agent 要做多步规划（选 query → 看结果 → 决定下一步 → 总结），思维链应该有用 → 开 `enable_thinking=True` 试一下。

## 实测对比（同一公司 Kole Imports / 同 prompt / 同 8 轮预算）

| 模式 | 命中条数 | 耗时 | 8 轮中产出有效 ACTION 的轮数 |
|---|---|---|---|
| **enable_thinking=False（默认）** | **5 条** | **60s** | 7/8 |
| **enable_thinking=True** | 4 条 | **3 分 29 秒** | 2/8 |

开了 thinking 反而：
- 条数下降（5 → 4）
- 年份精度下降（"1985-03" → "1990s / 2000s / 2020s" 模糊化）
- **6/8 轮 LLM 没产出 ACTION** — 写了大段推理但格式被搞坏，正则解析不到
- 总耗时 3.5×

## 根因

ReAct agent 的内核是「**严格输出格式 + 高频工具调用**」，不是「单次开放推理」。

- thinking 模式让 LLM 把推理写到 `<think>...</think>`（或类似），最终回应正文反而**缩短 + 跑偏**
- 越想越多，越容易脱离严格 `THOUGHT/ACTION` 模板
- 一轮 LLM 调用从 ~5s → 30-60s，8 轮直接爆炸
- 每轮多 thinking tokens（成本也涨）

类比：让人「先深度思考再说话」对单次提问是好的；但要他「按格式喊口令 8 遍」，深度思考反而让他口令乱掉。

## 适用 vs 不适用

| 任务类型 | 开 thinking？ |
|---|---|
| 单次开放推理（如选 KP / 写产业链分析 / 复杂判断） | ✅ 开 |
| 简单 JSON 抽取（如从段落抽公司名）| ❌ 关 |
| **ReAct / Tool-use agent loop**（N 轮严格格式 + 工具调用）| ❌ **关** |
| Research / Snippet 总结（multi-source 整合）| ❌ 关（实测同质量 -41% 成本，见下） |

## 横向印证

跟另一条踩坑同源：[qwen-thinking-off-research.md](https://github.com/zzhongster/trade-ai 的 memory)
— qwen3.6-plus 在 research / multi-source 总结任务上关 thinking，
实测**成本 -41% / tokens -36% / 质量同级**。

ReAct agent 是更严重的版本：**不仅成本贵 + 慢，质量直接退步**。

## 教训

- 「推理强 = 任务好」是错觉，要看任务**形状**：开放生成 vs 严格格式 + 高频调用
- agent loop 设计阶段就该选**格式遵守优先**的配置，thinking 留给最终总结那一步（如果有）
- 改实验性配置（如 thinking 开关）必须真跑对比，不能拍脑袋

## 实施细节

```python
# ❌ 错：ReAct agent 里开 thinking
for round_idx in range(MAX_ROUNDS):
    resp = await create_completion(
        model="qwen3.6-plus",
        messages=messages, max_tokens=1500,
        enable_thinking=True,  # ← agent loop 别开
    )

# ✅ 对：关 thinking，靠 prompt 显式列研究流程指引
for round_idx in range(MAX_ROUNDS):
    resp = await create_completion(
        model="qwen3.6-plus",
        messages=messages, max_tokens=1500,
        # 默认 enable_thinking=False
    )
```

## 相关

- 实测 commit: `c82eafe` → `a55f812`（trade-ai 仓 feature/web-react-rewrite）
- 文件: `server/app/services/milestones_agent.py`
- 同源 anti-pattern: qwen-thinking-off-research（research/JSON 抽取关 thinking 收益相反方向但本质同 — 都是「形状不匹配」）

## 日期

2026-05-25 trade-ai milestones agent v2 实测
