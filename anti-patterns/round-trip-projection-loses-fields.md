# 双向投影绕一圈丢字段

## 症状

新加了一个字段，写库时已经持久化，黑盒查 SQL 也能看到 — 但前端拿不到。怀疑前端 transform，怀疑 build 缓存，怀疑浏览器缓存…一圈下来根因是：**字段在「正向投影」加了，但「反向投影」没加，绕一圈数据就丢了**。

## 触发条件

后端做了双向数据投影时极易踩：

```
新数据 → 正向投影（forward） → 持久化（blocks/jsonb）
持久化 ← 反向投影（reverse） ← 前端读取（API）
```

经典场景：
- 内部新结构（blocks/事件流/规范化表）+ 旧 API schema（legacy data）的「兼容投影层」
- ORM model ↔ DTO ↔ API response 多层映射
- Protobuf ↔ JSON 互转

任何一边漏掉新字段都会沉默丢失（不报错，只是空）。

## 排查路径（按这个顺序，30 秒定位）

1. **SQL 直查**：`SELECT ... FROM table WHERE id=X` — 字段在不在持久化层？
   - 不在 → 写入路径（正向投影）问题
   - 在 → 继续
2. **API 直查**：`curl /api/.../X` — 字段在不在 API 响应？
   - 不在 → 读取路径（反向投影）问题 ← **大概率在这里**
   - 在 → 继续
3. **前端 transform 直查**：浏览器 DevTools → Network → 响应体 grep 字段名
   - 在 API 响应里有，但 transform 后没有 → 前端 transform 缺字段映射

**反过来排查**（先 grep 代码、先看 build、先看缓存）会浪费大量时间。

## 修复

每次给「正向投影」加字段时，**同一个 commit 顺手在反向投影也加**。两个投影函数最好放同一个文件邻近位置，让漏改时 diff review 一眼能看到。

更狠：写测试 `assert reverse(forward(x)) == x`（roundtrip 等价），自动捕获字段遗漏。

## 实例（trade-ai #250）

- 后端 `legacy_projection.py` 同时有 `legacy_to_blocks()`（写入）+ `blocks_to_legacy()`（读取）
- 加 `importer_totals` 字段时只改了写入侧
- 数据 → blocks（含 importer_totals）→ DB（含 importer_totals）→ blocks_to_legacy → API 响应（**丢失**）→ 前端「—」
- 排查浪费 ~10 min（先怀疑前端 transform，再怀疑 build 部署，最后才查反向投影）
- 修复：blocks_to_legacy 补一行 `"importer_totals": content.get("importer_totals") or {}`
