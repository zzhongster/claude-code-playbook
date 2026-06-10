# 表头/标签模糊匹配的子串方向陷阱

## 一句话结论

用"别名 in 表头"做字段列识别时，短别名会误吞长表头（"工厂" 命中 "工厂交期"）；反过来"表头 in 别名"又漏掉加长表头（"出货数量箱数" 配不上 "箱数"）。正解：两遍匹配——先全字段精确占列，再 contains 回退且跳过已占列。

## 症状 / 触发条件

Excel/CSV 有多种格式，列名会变但要素齐全，于是给每个字段配一组别名，读表头按名称定位列。匹配回退用子串时翻车：

- 写 `any(alias in header)`：字段"厂商"别名含"工厂"，结果把"厂商"错配到"工厂交期"列。
- 改成 `any(header in alias)` 想躲开：又让"出货数量箱数"这类加长表头匹配不上"箱数"——而这恰恰是要兜的真实场景。

两个方向各修一个、各坏一个，来回横跳。**更阴的是**：真实样例表头常常正好走"精确等于"路径，回退分支根本不触发，测试和 demo 全绿，把这个语义退化盖住了。

## 详细说明（正解）

两遍 + 精确占用列排除：

```python
claimed = set()
# 第一遍：精确等于别名，记录占用的列
for field, aliases in alias_dict.items():
    for col, header in enumerate(headers, 1):
        if norm(header) in [norm(a) for a in aliases] and col not in claimed:
            result[field] = col; claimed.add(col); break
# 第二遍：contains 回退（别名是表头子串），但跳过已被精确占用的列
for field, aliases in alias_dict.items():
    if result.get(field): continue
    for col, header in enumerate(headers, 1):
        if col in claimed: continue
        if any(norm(a) in norm(header) for a in aliases):
            result[field] = col; break
```

为什么有效：

- "工厂交期"列被"工厂交期"字段**精确占用** → "厂商"的"工厂"别名回退时跳过它 → 不再误配。
- "出货数量箱数"没被精确占用 → "箱数"别名 contains 命中它 → 加长表头照样识别。
- 保留了完整别名表（含"工厂"），不用为了躲冲突删别名。

附加：匹配前先 `normalize`（去空格 / 全角转半角 / 去尾部冒号 / 小写）。同样的思路也适用于"按标签定位写入单元格"——标签别名匹配同理。

## 数据支撑

凯越进仓单工具实测，3 个边界用例：

| 表头场景 | 别名⊆表头（无占用排除）| 表头⊆别名 | 正解（两遍+占用排除）|
|---|---|---|---|
| 无厂商列、有"工厂交期" | 厂商误配工厂交期列 ✗ | 厂商=None ✓ | 厂商=None ✓ |
| "出货数量箱数" | 箱数命中 ✓ | 箱数漏匹配 ✗ | 箱数命中 ✓ |
| "工厂"独立成列 | 厂商命中 ✓ | 厂商命中 ✓ | 厂商命中 ✓ |

教训：要专门构造**加长表头**和**冲突表头**的测试，别只用真实样例（会走精确路径，掩盖回退分支的 bug）。

## 适用范围

- 适用于：Excel 列识别、CSV 表头映射、表单标签定位，任何"别名 → 字段"的模糊定位。
- 不适用于：表头/标签完全固定、可直接写死坐标的场景（那就别上模糊匹配）。

## 相关

- `patterns/openpyxl-fill-xlsx-template-preserve-images.md`
- 首次踩坑项目：凯越进仓单自动生成工具（kaiyue-warehouse-tool），AI 实现时第一版就踩了"别名⊆表头"误配，审查才发现回退方向被改坏。
