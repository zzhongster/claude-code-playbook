# `_cat/indices` 的 docs.count 把 nested 子文档也算进去——对外报的总量虚增近一倍

## 现象

用 `GET _cat/indices?h=index,docs.count` 累加出"全库 49.9 亿条"，写进了产品备忘录和给上游的需求文档。
按 `_search {"size":0,"track_total_hits":true}` 逐索引精确计数后：**38.4 亿**。虚增的 11.5 亿全部来自
带 nested 字段的新一代索引——每条记录的 nested 子文档在 Lucene 层是独立 doc，`_cat` 照单全收。

同一口径错误还制造过第二个事故：以 `_cat` 数当分母算字段覆盖率，得出"2025 年 product_ids 覆盖 45%"，
而各市场整齐地都在 49% 上下——**跨市场异常一致的比率本身就是口径错误的指纹**（真实覆盖 93–99%，
分母大了一倍所以全部折半）。

## 机制

`_cat/indices` / `_stats` 的 `docs.count` 是 **Lucene segment 里的 doc 数**：nested 数组每个元素一个
隐藏子文档，父文档一个，全都计入。`_search` 的 `hits.total` 只数根文档。两者在无 nested 的索引上相等，
在有 nested 的索引上差出 nested 元素总数——而"部分索引有 nested、部分没有"的混合集群里，误差不均匀，
没法用一个系数修正。

## 判据

- 对外要报"多少条记录"，一律 `_search` + `track_total_hits:true`；`_cat` 只用来看索引存在性/健康度/磁盘。
- 两个口径对不上时先 `GET <index>/_mapping` 找 `"type": "nested"`。
- 各分组的比率异常一致（都≈50%、都≈33%）→ 先怀疑分母口径，不是数据。

## 成本

精确计数是每索引一次 `_search size:0`，593 个索引全量跑约 2 分钟——不是不做的理由。

## 实例

trade-agent-data，2026-08-29：商业化备忘录/回灌需求单里的"49.2 亿条/覆盖 19.1%"全部由此虚增，
修正为 38.4 亿/24.6%；nested 字段是提单索引的 `raws`（vn 2025 实测 `_cat` 1.72 亿 vs 精确 8,670 万）。
