# 反模式：按维度分片跑批时，分片谓词不是全集的划分，"全覆盖"不变量永远达不成

## 一句话结论

分片谓词必须是同一个规范化表达式的**互斥且穷尽**划分（`canonical = f(raw)`，正常分片 `canonical = X`，兜底分片 `canonical NOT IN 白名单`），并且下游所有按分片取数的地方都复用同一个谓词函数；否则总有一批行不属于任何分片，`sum(分片) = 全量` 这类不变量在最后一刻才发现过不去。

## 场景

- 大表按国家 / 日期 / 租户分片跑批，每片独立产出，最后要求"每一行恰有一条结果"
- 分片列脏：NULL、空串、大小写混杂、前后空格、非法值（`RSA`、`Unkn`、`9`、`/`、西里尔字母）
- 增强字段来自另一张表，也按"自己的"分片列抽取

## 踩坑经历

2026-08-29 企业样本去重项目（v6 表 1.38 亿行，按 `structured_country_code` 分片）。评审发现三处叠加的漏行：

1. 兜底分片写成 `cc IS NULL OR cc = '' OR LENGTH(cc) <> 2`——长度恰好为 2 的非法值（`9 ` 带空格、小写 `cn`、`XX`）既不进 CN 分片也不进兜底分片
2. 增强表 v4 按 **v4 自己的**国家码抽取，而不是按 v6 分片的 id JOIN——同一行两张表国家码写法不同时增强字段丢失
3. 运行脚本硬编码了 23 个国家，其余国家永远不跑

三处任何一处都让 `count(result) = count(v6)` 不可能成立，而 spec 里这条正是禁止回写的门槛。

## 正确做法

```python
def cc_predicate(cc: str, whitelist: frozenset[str], col: str) -> str:
    if cc != "XX":
        return f"upper(trim({col})) = '{cc}'"
    wl = ",".join(f"'{c}'" for c in sorted(whitelist))
    return f"({col} IS NULL OR upper(trim({col})) NOT IN ({wl}))"
```

- 三条规则：**一个规范化表达式**（`upper(trim())`）、**白名单否定式兜底**、**唯一谓词来源**（抽取、快照、增量、回滚全部调用同一个函数）
- 分片清单从数据里查（`SELECT canonical_cc, count(*) GROUP BY 1`），不硬编码
- 单测直接验证划分性质：往一列里塞 `["CN", " cn ", "us", "RSA", "9", "/", "", None, "Unkn", "XX"]`，断言各分片计数之和等于总行数
- 增强表按主表分片的 **id 集合** JOIN，不看增强表自己的分片列

## 识别信号

- 谓词里出现 `LENGTH(x) <> 2` / `x = ''` 这类"枚举脏值"的写法——枚举永远不完整
- 分片列表出现在 shell 脚本或 README 里
- 两张表各自 `WHERE country = ...` 再按 id 对齐
