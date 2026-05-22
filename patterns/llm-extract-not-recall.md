# LLM 不要凭记忆翻译/解析，让它从联网结果里抽

## 一句话结论

**对事实查询任务，LLM 凭记忆直接回答会音译瞎编，但读 Serper snippets 抽取又稳又准**。同一个事实，prompt 形式不同，准确率天差地别。

## 场景

任何需要 LLM 给出"准确事实"的任务：
- 英文品牌名 → 中文工商注册名
- 公司名 → 法人 / 统一信用代码 / 注册地
- 产品型号 → 厂家全称
- 历史事件 → 具体日期 / 地点 / 人物
- 任何"小众但客观存在"的对应关系

## 反讽点：同一事实，两种问法天差地别

trade-ai 真实场景：BVD 数据库收录中国企业用中文工商名（如 focallure 用「菲鹿儿」），英文别名 0 命中。需要 LLM 给出中文名作为 fallback 搜索关键词。

| 任务形式 | LLM 表现 | 准确率 |
|---------|---------|-------|
| 直接问「focallure 中文名是什么」(temperature=0) | 答 **菲奥拉露** | 0% — 音译瞎编最自信的字 |
| brainstorm「测哪些关键词可能命中」 | 偶发联想出 **菲鹿儿** | ~30% 看激活路径 |
| 看 Serper「focallure 公司名」结果 5 条 snippet 抽取 | 提取 **菲鹿儿** | 100% — 知名小众都救回 |

**为什么差这么大**：
- **直接翻译任务**激活的是「音译生成」模式 — LLM 选 token 概率最高的中文音译序列（菲+奥+拉+露），完全脱离事实
- **从 snippets 抽取**是阅读理解任务 — 不需要凭空记忆，只要从文本里找出来。LLM 的 reading comprehension 远比 recall 可靠

## 解决方案

**反查代替翻译**：

```python
# ❌ 错：让 LLM 凭记忆翻译
async def translate_to_cn(name):
    resp = await llm.complete(f"把 {name} 翻译成中文工商名")
    return resp.content
# focallure → 菲奥拉露（错）

# ✅ 对：Serper 反查 + LLM 抽取
async def resolve_cn_name(name):
    snippets = await serper.search(f"{name} 公司名", num=5)
    prompt = (
        f'下面是关于英文品牌 "{name}" 的搜索结果。'
        '请提取该品牌对应的中文工商注册名（如有），'
        '只输出一个中文公司/品牌名（2-8 字），不要解释。'
        f'如未明确提及则输出 NONE。\n\n{format_snippets(snippets)}'
    )
    resp = await llm.complete(prompt, temperature=0)
    return resp.content
# focallure → 菲鹿儿（对）
```

## 实测数据

trade-ai bvdid_resolver fallback 实测 5 个英文品牌：

| 输入 | LLM 直接翻译 | Serper + LLM 抽取 | 真实工商名 |
|------|------------|-----------------|----------|
| focallure | 菲奥拉露 ❌ | 菲鹿儿 ✓ | 菲鹿儿 |
| Florasis | 花西子 ✓ | 花西子 ✓ | 花西子 |
| Anker | 安克创新 ✓ | 安克创新 ✓ | 安克创新 |
| SHEIN | 希音 ✓ | 希音 ✓ | 希音 |
| AirPods | (N/A 不适用) | 苹果公司 ✓ | 苹果公司 |

**LLM 直接翻译只对知名 brand 准，小众必错**；Serper 反查 + LLM 抽取**全部命中**。

成本：+~5s Serper + ~1s LLM 抽取 = ~6s/次。对 0 命中兜底场景完全可接受。

## 推广

凡是需要 LLM 给「客观事实对应关系」的任务，先想：

1. **能否用搜索引擎反查**？能 → 用搜索 + LLM 抽取，别让 LLM 凭记忆
2. **prompt 是「生成」还是「阅读理解」**？生成模式（翻译/音译/补全）会瞎编，阅读理解（从给定文本抽取）远更可靠
3. **temperature=0 不能拯救记忆类任务** — 它只稳定瞎编的输出，不能让 LLM 知道它不知道的事

## 相关 pattern

- [[four-stage-search]] — 搜索调研的四阶段工作流
- [[ai-native-thinking]] — AI 原生思维

## 适用边界

不适用：
- 主观判断 / 风格生成（这些本来就不需要"事实准确"）
- 严格隐私 / 不能联网的场景
- 实时性要求高于 6s 延迟的场景

适用：
- 任何对外公开过、能被 Google 收录的事实查询
- 一次性调用 / 离线批量任务
- 准确率优先于延迟的兜底场景
