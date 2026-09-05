# 反模式：「纯 mock 单测」只是口头约定时，真实网络会藏在里面——症状是慢，换个网络就变成挂死

## 一句话结论

「单元测试不连外部依赖」若只写在文档里而没有**机制**，真实网络调用会悄悄溜进去；它平时只表现为「有点慢」因而被长期容忍（谁都不会为「测试跑 4 分钟」立案），直到换一个 DNS 行为不同的环境（沙箱把出网重定向到黑洞网段）才突然变成**挂死**。更坏的是：它可能已经让某些断言根本没在测它以为在测的东西——**网络调用发生在断言之前，测试因超时/异常走了另一条路，却仍然是绿的**。

## 场景

- 项目里有「单元测试是纯 mock、秒级」这类契约，写在 CLAUDE.md / CONTRIBUTING 里
- 某个构建函数（依赖注入的 `build_deps()`、`create_app()`、`register_tools()`）在**构造期**就做健康检查、版本对账、能力探测
- 测试通过一个「幂等守卫」间接调它：`if already_built: return` —— 而测试里的 `REGISTRY.clear()` / `reset()` 恰好把守卫打掉，于是每个用例都重跑一次探测
- 本地开发机 DNS 对不存在的主机名立即 NXDOMAIN，所以「慢」的量级是可以忍的几秒；CI 和沙箱不一定

## 踩坑经历

trade-agent-data（2026-09-05）：跑全量单测时，一个**专测 fail-fast 的用例**真的挂住了——不是慢，是挂。

```python
def build_default_deps() -> dict:
    es_url = _env("ES_URL")
    try:
        _live = live_index_names(es_url)          # ① 探测
    except Exception:
        _live = None
    ...
    return {
        "category_note": category_drift_note(es_url, ...),   # ② 探测，在字典字面量里
        ...
        "contact_source": ContactSource(_env("CONTACT_ES_URL")),  # ← 被测的 fail-fast 点
    }
```

测试的意图是「删掉 `CONTACT_ES_URL` 应该立刻 `RuntimeError` 点名该变量」。作者知道①会连网，把它 mock 了；**漏掉了②**。而②在返回字典的**字面量**里，求值顺序早于 `contact_source` 那行——**测试根本走不到断言点就先去连 ES 了**。

三层放大：

1. **幂等守卫被测试自己打掉**。`register_builtin_tools()` 里 `if "find_buyers" in TOOL_REGISTRY: return` 本可让探测只发生一次；测试里的 `TOOL_REGISTRY.clear()` 让 24 个文件的**每个用例**都重跑一遍。全量 3:47 里约 200s 在等网络，**CPU 只占 4%**。
2. **socket timeout 保护不了 DNS**。`urlopen(url, timeout=5)` 的 timeout 是 socket 超时；**`getaddrinfo` 不受它约束**。假主机名在普通开发机上立即 NXDOMAIN（所以只是慢），在把出网重定向到黑洞网段（如 `198.18.0.0/16`）的沙箱里会按解析器自己的重试策略阻塞——分钟级，且 timeout 参数完全无效。**同一个 bug，两种症状，其中一种被容忍了很久。**
3. **「慢」没有立案资格**。200s/次 × 每天几十次，没人会为此开卡；直到它变成挂死，才被当成 bug 看待。

## 解法

**1. 把契约落成机制，而不是逐个用例打补丁。**

在根 `conftest.py` 加 autouse fixture 短路探测。**放行按 marker 不按路径**——新测试真需要上游却忘打 `@pytest.mark.integration`，会在守卫这里炸出来，而不是静默变慢：

```python
@pytest.fixture(autouse=True)
def _no_upstream_probes(request, monkeypatch):
    if (request.node.get_closest_marker("integration")
            or request.node.get_closest_marker("quality")):
        return
    def _no_probe(url, timeout=10):
        raise OSError("单元/黄金套件不连上游")
    monkeypatch.setattr(registry, "live_index_names", _no_probe)
    monkeypatch.setattr(registry, "category_drift_note", lambda es, local, timeout=5: None)
    # 兜底：这条路径上将来新增的探测同样走它
    monkeypatch.setattr(introspect, "_urlopen_once", lambda req, timeout: pytest.fail(
        f"纯 mock 套件不该真连上游: {req}"))
```

**2. chokepoint 守卫要用 `BaseException` 才穿得透。**

有诚实降级设计的代码库里 `except Exception:` 遍地都是（数据源失败永不抛给用户）。用 `RuntimeError` 做的守卫会被**沿途任何一个降级 handler 静默吞掉**——守卫存在但从不出声，比没有守卫更糟。`pytest.fail()` 抛的 `Failed` 继承自 `BaseException`，能穿透所有 `except Exception:`。

**3. mock 的返回值要选「与现状同构」的那个，不要选「看起来更干净」的那个。**

`live_index_names` 的 mock 有两个候选：返回 `set()`，或者抛异常。返回空集看着更"正常"，但会让下游 `drift_note` 算出「528 项已失效」，**这条 warning 会透出到所有业务响应里**，可能改掉黄金测试的预期。抛异常则与「单测环境本来就连不上 ES」的既有行为完全一致（`_live=None` → `note=None`），**零行为变更**。

判据：先问「今天这行代码在测试环境里实际走的是哪条路径」，让 mock 复现那条，而不是复现"理想世界"。

**4. 别一刀切到最上层。**

同一模块里 `drift_note` 也在探测链上，但有测试**断言它返回非 None**。patch 到那一层会误伤。留给那些测试自己 `setattr` 覆盖即可——monkeypatch 后设的赢。

**5. 反验守卫真的会红。**

改**副本**而不是原地改原文件（破坏性测试要能扛中断），去掉一处 mock 跑一遍，确认守卫在正确的用例上炸出并点名了正确的 URL。守卫写完不拆一次，就分不清「测试存在」和「测试有效」（同 `guard-passes-without-the-fix`）。

## 数据支撑

trade-agent-data，1606 个非集成用例：

| | 修复前 | 修复后 |
|---|---|---|
| 全量 `-m "not integration and not quality"` | 227.66s，CPU **4%** | **7.80s**，CPU **75%** |
| 结果 | 1606 passed, 15 skipped | **1606 passed, 15 skipped**（逐项一致） |
| `test_registry.py` 单用例 | 15\~33s | 0.01\~0.02s |
| 最慢用例 | 33.13s（等网络） | 1.11s（真在算） |

**29 倍**，且零行为变更。CPU 占用从 4% → 75% 是最直接的判据：**CPU 空闲率就是「测试在等谁」的读数**。

## 怎么早点发现

- `pytest --durations=N`：单元用例出现秒级耗时就该问「它在等什么」，别接受"这个用例本来就重"
- `time` 跑全量看 CPU 百分比：**个位数百分比 = 全在等 I/O**，对纯 mock 套件是确定的异常信号
- 新增「构造期就做探测」的代码时（健康检查、版本对账、能力快照），当场问一句：谁会在测试里调到它

## 适用范围

- 适用于：任何有「构造期副作用」的依赖注入/注册表/app factory；有诚实降级（吞异常）设计的代码库尤其危险，因为网络失败不会变成红色测试
- 也适用于：DB 连接、消息队列、feature flag 拉取、license 校验——凡是 import/构造阶段就联网的
- 不适用于：集成/契约测试本身（那里连真库是目的）。所以放行口径要按 marker，别按目录

## 相关

- [测试全绿，但拆掉修复它还是全绿](guard-passes-without-the-fix.md) — 守卫写完要真去拆一次
- [tmp_path 测不到重跑污染](tmp-path-hides-rerun-pollution.md) — 测试环境模拟了一个不会发生的世界
- [指标绿的原因不是你以为的](green-metric-measures-wrong-mechanism.md) — 绿不等于测到了
- [吞 stderr 把缺工具伪装成空数据](silencing-stderr-hides-missing-tool-as-empty-data.md) — 同源：异常被吞掉后，失败与正常同形
