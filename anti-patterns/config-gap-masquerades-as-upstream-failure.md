# 反模式：配置缺失被裸 except 吞成"上游故障"——契约文案一个字没改，发出它的条件却变宽了

## 一句话结论

当一句降级文案同时是**机器判据**（计费豁免 / 下游熔断 / 告警分类）时，真正的风险往往不是有人改了这句文案，而是**发出它的那条 `except` 覆盖面比文案描述的条件宽**。于是"我方漏配一个环境变量"继承了"上游数据源挂了"的全套语义：免掉不该免的费、对下游发出错误的熔断信号、让监控对着一个纯配置问题告警——三个动作彼此自洽，从外面看不出任何异常。

这类改动**不会在 diff 里碰到那句契约文案**，所以评审天然看不见。

## 场景

- 系统有"诚实降级"约定：数据源失败永不抛给用户，返回空结果 + 一句统一 warning
- 那句 warning 被程序消费：`if "查询失败" in warnings` 用来免计费、下游按前缀熔断、监控按它统计数据源健康度
- 降级分支写成 `except Exception:`（为了兑现"永不抛"的承诺，这几乎是必然写法）
- 存在一类**不是上游故障**的异常，也会从同一个调用里冒出来——配置缺失是最常见的一种

## 踩坑经历

trade-agent-data（贸易数据 MCP，2026-09-06）。本地 `.env` 缺 `EMAIL_VERIFY_URL` 时，`verify_email` 返回：

```json
{"data": [], "meta": {"returned": 0, "verified": 0, "query_time_ms": 0.0},
 "warnings": ["邮箱数据源查询失败，已返回空结果"]}
```

真实异常是：

```
ValueError: unknown url type: 'None/v1/oauth/access_token'
```

代码里是一行再普通不过的拼接：

```python
resp = self._request("POST", f"{self._verify_url}/v1/oauth/access_token", ...)
```

`self._verify_url` 为 `None` 时，f-string 拼出**字面量字符串** `"None/v1/oauth/access_token"`，直到 `urllib.request.Request` 构造时才抛 `ValueError`。工具层的 `except Exception:` 把它和真正的网络故障一视同仁，吐出同一句降级文案。

### 三个连锁后果

| 机制 | 本该表达 | 实际发生 |
|---|---|---|
| 计费豁免（扫 warnings 含"查询失败"→ 免扣 + 写 `__degraded__` 审计行） | 上游挂了，不该收客户钱 | **我方漏配环境变量，免掉了客户的费** |
| 下游熔断（TOUCH 按 warnings 判定数据源不可用） | 数据源抖动，退避/熔断 | **对下游谎报"数据源挂了"** |
| SLA `source-watch`（按账本 `__degraded__` 行统计各工具降级/成功比，连续无成功即判 dead 开单） | 数据源持续故障告警 | **对着一个纯配置问题开故障单** |

三个动作互相印证，看起来完全自洽——这正是它难被发现的原因。响应里唯一的破绽是 `query_time_ms: 0.0`（压根没发出网络请求），而这需要有人恰好盯着它并且知道它意味着什么。

## 两个让它隐形的机制（可迁移）

### 1. 缺失的配置不会 TypeError，它会生成"语法合法但语义荒谬"的值

`f"{None}/path"` 不报错，产出 `"None/path"`。真正的失败推迟到下游某个消费者，**且失败形态与运行时故障同形**：

- URL 拼接 → `ValueError` / 连接错误 → 长得像网络故障
- 路径拼接 → `FileNotFoundError` → 长得像文件被删
- SQL / 查询条件拼接 → 空结果 → 长得像"真的没数据"

**推论：事后按异常类型分辨是徒劳的**，因为异常类型已经被同化了。判定必须前置到"生成那个荒谬值之前"。

同族的 Python 陷阱：`os.environ.get("X")` 返回 `None` 而不抛，和 `f-string` 一配合就是这个形态。项目里凡是**不走 fail-fast 的可选配置**，都是候选点。

### 2. mock seam 的位置决定了哪一段代码**任何测试都跑不到**

该数据源的单测约定是"只 mock `_request` 这个 HTTP 基元，其余走真实代码"——听起来很扎实。但 bug 恰好在 `_request` **调用参数的构造**上：

```python
def _verify_token(self):
    if not (self._verify_id and self._verify_secret):     # ← 只查了凭证
        raise RuntimeError("...not configured")
    ...
    self._request("POST", f"{self._verify_url}/v1/oauth/...")   # ← 没查地址，洞在这
```

所有用例都从 mock 掉的 `_request` **之后**才开始断言，于是"URL 是 `None` 却照样往下走"这一段**从未被任何测试执行过**。20+ 条该模块的单测全绿，一条也抓不到。

seam 之上有一条盲带，宽度就是"从入参构造到 seam"那几行。写测试时值得问一句：**我 mock 的这一层，把哪几行代码永久排除在覆盖之外了？**

## 正确做法

### 1. 判定前置，别指望事后分类

给配置缺失一个**专用异常**，在发请求之前抛：

```python
class EmailConfigError(RuntimeError):
    """我方配置缺失（上游地址/凭证未配），不是上游故障。"""

def _verify_token(self):
    if not (self._verify_url and self._verify_id and self._verify_secret):
        raise EmailConfigError("verify endpoint/credentials not configured")
```

关键是**地址和凭证一起判**。原代码只判凭证——"有 key 就算配好了"是个很自然但错误的直觉。

### 2. 专用异常必须排在裸 except 之前

```python
try:
    rows = source.domain_emails(d)
except EmailConfigError:
    raise RuntimeError(_NOT_CONFIGURED) from None    # 新的对外通道
except Exception:
    return degraded_response()                        # 原有诚实降级
```

顺序写反 = 白改。

### 3. 想清楚新通道的计费/告警语义，别复用旧的

这次选了**抛异常**而不是返回降级响应，因为该框架的执行顺序是 `限流 → 授权预检 → 余额预检 → 执行 handler → 扣费`：handler 抛出 ⇒ 扣费根本不执行 ⇒ **不计费、账本零行**（连 `__degraded__` 审计行都不留），与"授权未开通"的预检拒绝同语义。

另一条路（返回 200 + 业务口径 warning）会**照常计费**——等于让客户为我方的配置缺失付钱，比原 bug 好不了多少。**"不走故障豁免"不等于"该收钱"，这两件事要分开决定。**

顺带一个免费的正确性：判定放在 handler 里，天然排在授权预检**之后**，所以未授权的调用方拿到的是"该功能需单独申请开通"，看不到我方的配置状态。这条值得单独写一个守卫钉住——万一哪天有人把配置判定"优化"成前置预检，就会泄露出去。

### 4. 按能力分组判定，别一刀切

三个工具依赖的变量各不相同（`EMAIL_VERIFY_*` vs `EMAIL_SCRAPE_*` vs `EMAIL_OPENAPI_*`）。一刀切成"邮箱功能未配置"会让"验证没配"连累"挖邮箱"一起不可用。

### 5. 新的对外文案 = 新的契约面

`"该功能暂不可用"` 落地时要同步：写进给下游的契约文档（错误分类表 + 稳定文案清单）、写进项目 CLAUDE.md 的红线章节、通知集成方。哪怕是纯增量也要说——集成方通常有"未知前缀 → 熔断"的兜底，你至少得告诉他们这个状态不会自愈、退避重试无意义。

## 自查清单

在自己的项目里找同类洞，三个问题：

1. **哪些 warning / 错误文案被程序消费？** 找 `if "xxx" in warnings`、集成文档里的"按前缀识别"。每找到一条，去看**发出它的那个 `except` 覆盖了哪些异常来源**。
2. **哪些配置不走 fail-fast？** 本项目实测：`registry.py` 里 5 个数据源，4 个走 `_env()`（缺变量启动即失败），只有邮箱源用裸 `os.environ.get`——洞就**只**在这一个。这个排查花了两分钟，结论是"其余无需再查"，很划算。同时它也界定了另一类问题的边界：走 `_env()` 的那批只会以"启动失败"形态出现，不会以"半套配置跑起来"形态出现，两者互斥。
3. **可选配置里，哪些字段只查了一半？** 典型是"查了凭证没查地址"、"查了 host 没查 port"、"查了开关没查开关打开后必需的那几个字段"。

## 数据

- 修复配套 7 条用例，撤掉修复后**全部转红**（含端到端计费语义那条：断言账本零行）
- 用例故意用真实 `EmailSource` 而非 stub —— 用既有的 stub 风格写，这 7 条会全绿（同"mock seam 盲带"那节）
- 全量 1617 passed / 15 skipped；黑盒扫描 16 passed
- 该项目 2026-09-03~04 曾发生上游验证服务断供，**自动化零告警**，靠真机撞上才发现——与本条是同一个盲区的两面：一边是真故障没告警，一边是假故障在告警

## 关联

- [一旦下游用文案做程序判断，人类可读文案就是 API](../patterns/user-facing-copy-is-machine-contract.md) —— 本条是它的**补集**：那条讲"别改契约文案"，本条讲"别扩大发出契约文案的那条分支的覆盖面"。文案没动、条件漂了，diff 里看不见，更隐蔽。
- [吞 stderr 把缺工具伪装成空数据](silencing-stderr-hides-missing-tool-as-empty-data.md) —— 同一族："错误被伪装成合法的观测结果"。那条在 shell 诊断层，本条在应用降级层。
- [测试全绿，但拆掉修复它还是全绿](guard-passes-without-the-fix.md) / [TDD 假 RED](tdd-fake-red.md) —— 本次的 7 条用例用了它们的 revert-FAIL 验证法；还原动作放在 `finally` 里，不写成末尾一条命令（超时中断会留下污染文件）。
- [纯 mock 单测里藏着真实网络](unmocked-network-in-pure-mock-tests.md) —— 同样是"mock 边界没想清楚"，方向相反：那条是外部依赖漏进来，本条是自己的代码被挡在覆盖之外。

（来源：trade-agent-data 2026-09-06，Linear YC3-47 同族。）
