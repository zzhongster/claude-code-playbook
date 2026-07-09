# 模式：计划里的参考代码不是圣旨——实现层要带脑子转录，偏离要走"反抓→trace→核准"闭环

## 一句话结论

写实现计划（writing-plans）时给出的完整参考代码会带着计划作者的 bug 一起下发；实现 subagent 的正确姿态不是逐行转录，而是**先理解控制流再落地**——发现参考代码有缺陷时，正当偏离 + 在报告里说明理由，由 task reviewer 逐态 trace 核准。这条链路真实抓过计划层 bug，是 SDD（subagent-driven-development）分层质检的价值样本。

## 场景

- 计划/spec 里包含"抄这段"级别的完整参考实现（SDD 的 task brief 常如此，转录型任务甚至配最便宜的模型）
- 参考代码含异常处理、控制流分支等"看起来对、状态机上错"的结构
- 实现者被要求"exact values verbatim"，容易把"照抄数值"泛化成"照抄一切"

## 踩坑经历（正面样本）

trade-agent-data S11 transport 计费集成（2026-07-09）：计划的参考代码在"执行成功后扣费"段写了

```python
try:
    new_balance = store.charge(...)
except RuntimeError:
    raise          # 意图：余额不足往外抛
except Exception:
    ...fail-open 放行
```

意图是区分"业务拒绝（往外抛）"与"基础设施故障（fail-open 放行）"。但 `except RuntimeError: raise` 会把**被包装成 RuntimeError 的 infra 故障也往外抛**，fail-open 分支形同虚设——计划作者自己的状态机 bug。实现 subagent（sonnet）没有照抄：识别出"charge 返回 None=业务拒绝、charge 抛异常=infra 故障"本可以用返回值区分，重构为 try/except/else——`else` 分支处理 None→抛余额不足，`except` 只接真正的 infra 异常→fail-open。报告里明确标注了偏离与理由。task reviewer 对四条路径逐态 trace 后核准；opus 整分支终审再次独立确认"这正是参考代码 bug 的正解"。

## 解法（角色分工）

1. **计划作者**：参考代码里的异常处理/控制流当成真代码 review 一遍再下发；"意图注释"（这段想区分什么）比多写防御代码更有用——实现者靠意图判断偏离方向。
2. **实现者**：brief 里 verbatim 约束的是**数值/命名/文案/接口签名**，不是控制流。落地前先在脑内跑一遍参考代码的状态机；发现缺陷 → 按意图重构 → 报告里写"偏离点 + 参考代码会怎么错 + 我的等价性论证"。
3. **task reviewer**：对声明的偏离不是"与 brief 不符"一票否决，而是逐态 trace 两个版本的行为差，确认偏离修的是真 bug。
4. **控制器**：把"实现者可以且应该质疑参考代码"写进 dispatch 契约；偏离未声明才是缺陷。

## 教训

- SDD 的价值不在"每层都听话"，在**每层都带独立判断**：实现层抓计划 bug、评审层验证抓得对不对、终审层再确认——三层里任何一层照章办事都会把 bug 带上线。
- "转录型任务用最便宜模型"的前提是参考代码经过了 review；含异常处理/并发/状态机的参考代码别当纯转录下发。
