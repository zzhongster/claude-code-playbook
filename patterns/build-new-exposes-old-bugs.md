# Build-New Exposes Old Bugs（搭新东西主动暴露老 bug）

> 在搭一个新功能 / 编排层时，用真实业务数据跑端到端 → 看 logs → 发现并修被掩盖的旧 bug。Compounding engineering 的高 ROI pattern。

## 典型案例

**场景**：trade-ai 2026-05-31 #290 epic — 搭 mini pipeline orchestrator（串联 3 个联网类小 Agent endpoint），验证整体设计可行性。

**步骤**：
1. ✅ 搭 orchestrator skeleton（`auto_research_pipeline_mini.py`）
2. ✅ 加 endpoint SSE 透传
3. ⚡ **用真实公司跑**（Mini DB 推荐里挑 STIGA Group，欧洲最大割草机厂）
4. 👀 看 docker logs：`[web_research] JSON 模式不匹配: text='[\n{\n"name":"STIGA S.p.A."...`
5. 💡 顿悟：LLM 输出被 max_tokens=1200 截断
6. 🔧 修 mini pipeline 的 max_tokens (1200→3500) → STIGA 拿到 30 条 offices
7. 🎯 反向修原 endpoint `fetch-offices` 的同 bug（影响所有用户点「🌐 联网查找」拿大集团的场景）

**ROI 评估**：
- 搭 mini pipeline 工作量：1 小时
- 实战发现的旧 bug 影响：所有用户对大型集团联网查询都返 0 条（之前没人报，因为：要么用户用的是小公司、要么默认接受了"LLM 没找到"）
- 修复成本：5 行代码
- 价值：解决了潜在的高频用户痛点

## 为什么这个 pattern 高 ROI

1. **新功能本身就是真实业务的代理**：搭编排层 = 把所有底层 endpoint 都过一遍真实数据
2. **logs 是免费的审计员**：你已经在 debug 新功能，顺手看到旧 bug
3. **修旧 bug 的 PR 价值往往 > 新功能本身**：影响面更广、用户感知更直接
4. **避免 "新功能上线 + 旧 bug 单独某天爆"** 的双重事故

## 怎么主动触发

写新编排层 / 集成层 / wrapper 时强制：

1. **必跑真实公司 / 真实数据**（不要满足于 mock 单测）
2. **跑前看一眼 docker logs 是否有 warning / error**（之前可能也在但没人看）
3. **每步打印关键统计**（如返回 0 条 vs 30 条 → 这是 bug 信号）
4. **diff 直接调 vs 经编排调的结果**（直接调成功 + 编排调失败 → 编排有 bug；直接调也失败 → 底层有 bug，搭新东西帮你发现了）

## 反例（什么时候不适用）

- 只调用稳定的内部 service（不联外网 LLM / 不调第三方 API）— 没新信号
- 新功能完全 mock 跑（如纯前端 component 重构）— 见不到真实数据 → 见不到 bug

## 相关 pattern

- [code-audit-checklist](./code-audit-checklist.md) — 写完代码主动审计
- [agent-loop-architecture](./agent-loop-architecture.md) — Agent 串联模式
