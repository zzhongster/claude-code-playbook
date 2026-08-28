# 反模式：派生缓存只检查文件是否存在，上游数据或算法变了照样复用

## 一句话结论

任何"由 A 算出来的 B"缓存，判定复用的依据必须是 **A 的内容版本 + 生成 B 的算法版本**，两者都要写进 B 旁边的 manifest；`if path.exists(): return path` 是在给自己埋一个只有出结果后才会发现的错。

## 场景

- 归一化 / 索引 / 嵌入等预处理结果落盘复用（本例：企查查英文名归一表、orbis 1.2 亿名称归一目录、名称→bvdid 命中表）
- 上游是数据库表，会被**原地更新**而不是追加（行数不变、`max(update_time)` 也可能不变）
- 归一词典、blocking 规则会迭代，改完想"只重跑后半段"

## 踩坑经历

2026-08-29 同一项目的实施计划评审：

- `build_qcc_norm` / `_orbis_norm` 都写成 `if out.exists(): return out`——词典改了、企查查表换了，S2.5/S4 仍吃旧文件
- `lib_manifest` 用 `count + max(load_time)` 和 `count + max(update_time) + sum(is_del)` 当版本——原地修正一条英文名，两个指标纹丝不动
- `qcc_bvdid_fix` 的摘要用 `MD5(GROUP_CONCAT(...))`——`GROUP_CONCAT` 有长度上限，6.5 万行会被静默截断，摘要只反映前一小段

## 正确做法

1. **内容摘要，顺序无关、无长度上限**：

```sql
SELECT COUNT(*), SUM(CAST(murmur_hash3_32(CONCAT_WS('|', id, name, bvdid, city, status)) AS BIGINT))
FROM dim.qcc_advanced_search
```

   每张上游表一个 `{count, digest}`，全部 JSON 后再哈希成 `lib_version`；词典文件 + 算法源码哈希成 `pipeline_version`。

2. **sidecar manifest**：

```python
def is_valid(artifact, lib_version, pipeline_version) -> bool:
    m = json.loads((artifact + ".manifest.json").read_text())   # 不存在即 False
    return m["lib_version"] == lib_version and m["pipeline_version"] == pipeline_version
```

   两个版本**全等**才复用；写缓存时同步写 manifest。目录型缓存（分批 parquet）manifest 放目录旁。

3. 版本号同时写进最终结果表的每一行（`lib_version`、`pipeline_version` 列），增量任务据此判断"要不要全量重算"。

## 识别信号

- 代码里出现 `if path.exists(): return`
- 版本用 `count(*)`、`max(时间列)` 拼——它们只能发现追加，发现不了原地修改
- 用 `GROUP_CONCAT` / `string_agg` 算大表指纹
