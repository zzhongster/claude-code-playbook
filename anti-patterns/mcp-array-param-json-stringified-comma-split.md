# 反模式：MCP 工具的 array 型参数被调用方字符串化后，naive 逗号切分产生畸形数据、静默丢失真实结果

## 一句话结论

MCP 工具的 `input_schema` 为了兼容"简化输入"常把某个参数声明成 `type: ["array", "string"]`（数组或逗号分隔字符串都收）；但某些 MCP 客户端/Agent 框架在构造调用时会把真实数组**序列化成 JSON 字符串**再传（而不是传真数组）。服务端如果对字符串输入无脑按逗号切分，会把 `'["a@x.com", "b@x.com"]'` 拆成带 JSON 语法字符（`["`/`"`/`"]`）的畸形 token——而且这些畸形 token 往往还能通过过松的本地格式校验，被送去下游继续处理，最终返回"格式合法但语义全错"的结果，调用方完全无法察觉。

## 场景

- MCP/Function-calling 工具的某个参数被设计成"数组或字符串都行"，图的是兼容简单调用场景（比如逗号分隔的邮箱列表）
- 调用方是某个 Agent 框架/客户端，其内部会把 tool_call 的 arguments 先序列化成 JSON 再反序列化一次，或者在某个中间层把 array 类型的字段错误地当成字符串处理
- 结果就是服务端收到的不是 `["a@x.com", "b@x.com"]`（真数组），而是字符串 `'["a@x.com", "b@x.com"]'`（数组的 JSON 字面量文本）

## 踩坑经历

2026-08-28，trade-agent-data 项目的 `verify_email` 工具（邮箱可达性批量验证）收到一份第三方 FDE 交付流程实测报告：批量验证 10 个真实有效邮箱，返回 `{"valid": 0, "invalid": 10}`——全部误判为 invalid，且返回结构完全合法（`status` 是枚举内正常取值），日志无任何异常。下游的交付规则是"invalid 直接清空不交付"，这一批导致 96 条真实采购决策人邮箱被静默判死删除。

对照实验一眼看出规律：同一个地址单个传入 `{"emails": "m.turan@ppc-ag.de"}` 完全正常；批量传数组就全错。返回的 `email` 字段本身就带着线索：

```json
{"email": "[\"m.turan@ppc-ag.de\"",        "status": "invalid"}
{"email": "\"j.wagner@ppc-ag.de\"",         "status": "invalid"}
{"email": "\"gorka@zivautomation.com\"]",   "status": "invalid"}
```

首元素带前导 `["`，末元素带尾随 `"]`，中间元素带首尾 `"`——典型的"字符串被按逗号切开"特征。本地复现：

```python
emails = '["m.turan@ppc-ag.de", "j.wagner@ppc-ag.de", "gorka@zivautomation.com"]'
[e.strip() for e in emails.split(",") if e.strip()]
# ['["m.turan@ppc-ag.de"', '"j.wagner@ppc-ag.de"', '"gorka@zivautomation.com"]']
```

完全吻合。

## 根因（两层叠加，缺一不可才会酿成"静默"而非"报错"）

1. **入参解析**：`type: ["array", "string"]` 的字符串分支，只按"逗号分隔的简单字符串"这一种意图设计，没考虑调用方可能传入的是数组的 JSON 序列化文本。
2. **本地格式校验过松**：项目里原来的邮箱正则 `^[^@\s]+@[^@\s]+\.[^@\s]+$` 不排除方括号/引号字符，畸形 token（如 `["m.turan@ppc-ag.de`）能通过本地校验，被当作"看起来合法"的数据继续往下游送——如果本地校验够严格，至少会在入口处显式失败，而不是让脏数据一路走到底、最后呈现成一个语义错误但格式合法的结论。

两层任一收紧都能避免"看起来合法但语义全错"这种最危险的失败模式；只做其中一层，另一层仍是隐患。

## 修复

字符串入参先判断是否形如 JSON 数组（`.strip()` 后以 `[` 开头 `]` 结尾），优先尝试 `json.loads` 恢复出干净的 `list[str]`；解析失败或元素不全是字符串，才退回原来的逗号切分（保留对"真·简化字符串输入"场景的兼容）：

```python
def _split_array_arg(raw: str) -> list[str]:
    s = raw.strip()
    if s.startswith("[") and s.endswith("]"):
        try:
            parsed = json.loads(s)
        except (json.JSONDecodeError, ValueError):
            parsed = None
        if isinstance(parsed, list) and all(isinstance(x, str) for x in parsed):
            return parsed
    return [e.strip() for e in s.split(",") if e.strip()]
```

同时把本地格式校验收紧（排除方括号/引号等 JSON 语法字符），作为第二层防护——即便某种边缘情况下解析仍有残留，也不会让畸形 token 蒙混过关去消耗一次真实的下游调用。

## 验证方法

TDD 补三类测试：① 真实数组入参（回归，确认没改坏）；② 逗号分隔字符串入参（回归，确认简化输入形态不受影响）；③ 数组被 JSON 字符串化后传入（新增，这是本次要修的场景，断言结果与①一致）。任何一个 array-or-string 型参数的 MCP 工具都该补这条③类测试，不要只测"正常数组"和"正常逗号串"两种理想输入。

## 适用范围

- **适用于**：任何声明 `type: ["array", "string"]`（或类似"多形态兼容"）的 MCP/Function-calling 工具参数，尤其是调用方可能来自不完全受控的第三方 Agent 框架时
- **不适用于**：参数类型收窄为纯 `array` 且服务端做了严格 schema 校验（拒绝非数组输入）的场景——那样从源头就不会进到这个坑，但代价是失去了对"简化字符串输入"调用方的兼容性，需要权衡
- **更通用的教训**：多形态兼容参数的字符串分支，不能只按"最简单的那种输入意图"设计；要假设字符串里可能装着别的形态序列化后的残留（JSON、逗号+空格混排、URL-encoded 等），做形态探测再决定怎么解析，而不是无脑套用一种切分规则

## 相关

- 项目源：`trade-agent-data`，issue #39，PR #43（`src/trade_agent_data/tools/email_tools.py: _split_email_arg`）
- 故障发现时间：2026-08-28（第三方 FDE 交付流程实测报告）
