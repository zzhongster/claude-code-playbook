# 反模式：`int(float(x))` 的 except 只捕 (TypeError, ValueError)——`"inf"` 抛 OverflowError 穿透

## 一句话结论

对脏输入做 `int(float(x))` 数值归一时，`float("inf")` **成功**、随后 `int(inf)` 抛的是 **OverflowError**（ArithmeticError 子类），不在惯性写的 `except (TypeError, ValueError)` 里——`"inf"/"-inf"/"Infinity"/"1e999"` 形态的脏值会穿透守卫直接崩掉调用链。而 `float("nan")` 走 `int(nan)` 抛 ValueError 被捕获——**inf 崩、nan 不崩的不对称**让这个洞在测试里格外隐蔽（用 nan/"abc" 测过全绿，inf 没测就是漏的）。

## 场景

- 清洗外部数据（ES `_source`、CSV、用户输入）里的"应为整数"字段：员工数、limit、计数
- 惯性防御写法：`try: v = int(float(raw)) except (TypeError, ValueError): 降级`
- 单测用 `"abc"`（ValueError）、`None`（TypeError）验证过守卫→绿→上线

## 踩坑经历（同一项目一周内踩两次）

trade-agent-data（2026-07）：

1. **S6 `search_companies`**：ES 企业库 `staff_count` 脏值守卫 `except (TypeError, ValueError)`——task reviewer 首轮只发现 `"N/A"`（ValueError，已捕获），复审才指出 `"inf"` 形态穿透，且该行在原语的 try/except 兜底**之外**的列表推导里，一穿透就击破"原语层永不抛"的项目红线。补 `OverflowError` + `staff_count="inf"` 单测。
2. **S9 transport limit 钳制**：`int(arguments["limit"])` 又写了 `except (TypeError, ValueError)`——opus 终审再次抓到 `limit=Infinity` 穿透成 500。同一个人（同一个 AI 流水线）一周内在两处独立代码写出同款洞。

## 解法

1. **元组补全**：`except (TypeError, ValueError, OverflowError):`——或语义就是"转不出干净整数就降级"时直接 `except Exception`（更贴意图）。
2. **测试三件套**：脏值守卫的单测至少覆盖 `None`（TypeError）、`"abc"`（ValueError）、**`"inf"`（OverflowError）**三类；只测前两类的绿灯是假安全。
3. **模式识别**：代码里出现 `int(float(` 就条件反射查 except 元组——这是可 grep 的机械检查（`grep -rn "int(float(" | 对照各自 except`）。

## 教训

- Python 数值转换的异常面不止 TypeError/ValueError：`int(float(...))` 有 OverflowError，`Decimal` 有 InvalidOperation——"惯性防御元组"要按实际调用链核，不按肌肉记忆写。
- 同款洞会复发：第一次修补只修了实例没沉淀模式（S6 修完没有全仓 grep `int(float(`），一周后同一模式在新代码再现。修守卫类 bug 时顺手全仓扫同模式。
