# 反模式：ES keyword 字段带 lowercase normalizer 时，把"聚合/过滤看到的小写"当成"存储的小写"

## 一句话结论

Elasticsearch 的 keyword 字段配了 lowercase normalizer 后，**terms 聚合桶和 term/terms 过滤都按小写归一工作，但 `_source` 返回的是文档原文**（可能是 Title-Case 或任意大小写）。如果你用聚合结果里的小写值建了解码字典（码→中文标签之类），再拿 `_source` 原文去查，就会静默 miss——过滤全对、数据全在，唯独展示层泄出未解码的原始值。纯 mock 单测永远抓不到这种"过滤对、解码错"的错配，必须真数据集成测试。

## 场景

- 对 ES 做数据勘探：用 `terms` 聚合看某 keyword 字段的值分布，把桶 key 记进 spec/字典
- 之后代码里两处用这份认知：① 过滤（term/terms 查询）② 解码（`_source` 值→人类可读标签的 dict 查找）
- 单测全用 fake 数据（按勘探记录的小写值造 doc）→ 全绿
- 上真实集群：过滤命中正常，但输出里的标签是未翻译的英文原文

## 踩坑经历

2026-07-08，trade-agent-data 的 `find_contacts`（6770 万 LinkedIn 联系人，ES 6.4）。勘探时聚合 `job_level.keyword` 得到分布：

```
other 45206884 / manager 13042492 / director 4137591 / c-level 3833477 / ...
```

全小写——于是 spec 记了 `DECISION_LEVELS = ["c-level", "vp", "director", ...]`（过滤用）和 `LEVEL_CN = {"director": "总监", "c-level": "C级高管", ...}`（解码用）。单测用小写 fake doc，14/14 绿。

golden 真集群一跑，`find_contacts("BMW", country="gb")` 断言 `seniority in {"总监", ...}` 失败——返回的是 **"Director"**（Title-Case 英文原文）。诡异之处：小写 terms 过滤 `["director", ...]` 明明命中了 28 条。

查 mapping 才破案：

```bash
$ curl -s localhost:9209/linkedincontactdata_v2/_mapping | python3 -c "..."
job_level: {"type": "text", "fields": {"keyword": {"type": "keyword", "normalizer": "my_normalizer"}}}
```

`my_normalizer` 做 lowercase。链路上三个视角看到的值各不相同：

| 视角 | 看到的值 | 为什么 |
|---|---|---|
| terms 聚合桶 | `director`（小写） | 聚合在归一后的 keyword 值上 |
| term/terms 过滤 | 小写、Title-Case 都命中 | query-side 也过 normalizer |
| `_source` | `Director`（原文） | `_source` 永远回存储原文，不归一 |

勘探只看了第一个视角，就把"小写"错误外推到了第三个视角。

## 解法

1. **解码字典查询前自己归一**：`LEVEL_CN.get(str(raw).strip().lower(), raw)`——一行修复。
2. **过滤入参也顺手客户端归一**（如 `job_function.strip().lower()`）：虽然服务端 normalizer 会兜住，但客户端归一把行为正确性从"索引 mapping 恰好配了 normalizer"这个隐性依赖里解耦——索引重建丢 normalizer 时不会静默回归。
3. **勘探时补一步交叉验证**：聚合看完分布后，随手 `size:3` 拉几条 `_source` 原文对比大小写形态；同一字段用 Title-Case 值发一条 term 查询，命中数≠0 即说明有 normalizer。
4. **必须有真数据 golden**：这类错配的两半（过滤、解码）各自都"对"，只有拼在真实数据上才碎。mock 单测按你的错误认知造数据，只能验证你的错误认知自洽。

## 教训

- 聚合/过滤是**归一后的世界**，`_source` 是**原文的世界**——任何"从 ES 值到本地字典"的映射，键的形态必须以 `_source` 实测为准，不能以聚合桶为准。
- 勘探记录里应显式标注"该字段 keyword 带 normalizer（聚合显示≠存储原文）"，否则下一个薄片还会踩（同一集群 `job_function.keyword` 也带同款 normalizer）。
