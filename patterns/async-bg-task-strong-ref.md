# asyncio.create_task fire-and-forget 必须保留强引用

## 核心原则

> Python 3.11+ 的 `asyncio.create_task()` 返回的 task 如果不被强引用，事件循环空闲时可能被 GC 提早回收，"任务消失"。

冷路径 fire-and-forget（写入完毕、消息发出后不需要结果的后台任务）最容易踩这个坑。

## 反模式：直接 create_task 不留引用

```python
def schedule_enrich(*, user_id: int, company_name: str):
    """onboarding 完成后异步触发 IdentityEnricher，不阻塞用户响应。"""
    asyncio.create_task(_bg_enrich(user_id, company_name))
    # ⚠️ 返回值丢弃 → Python 3.11+ 文档警告 task 可能被 GC
```

[Python 3.11 docs](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) 原文：

> Important: Save a reference to the result of this function, to avoid a task disappearing mid-execution. The event loop only keeps weak references to tasks. A task that isn't referenced elsewhere may be garbage collected at any time, even before it's done.

冷路径触发后主流程立即返回 → 事件循环 idle → GC 跑 → task 提早结束。

## 正模式：模块级 set + done_callback 自动出栈

```python
import asyncio

# 模块级别强引用集合
_BG_TASKS: set[asyncio.Task] = set()


def schedule_enrich(*, user_id: int, company_name: str):
    """触发后台 IdentityEnricher。fire-and-forget 但任务不会被 GC。"""
    task = asyncio.create_task(_bg_enrich(user_id, company_name))
    _BG_TASKS.add(task)
    task.add_done_callback(_BG_TASKS.discard)
    # ✅ task 完成后自动从 set 出栈，避免内存泄漏
```

三步：
1. **模块级 `set` 持有引用**（不能用局部变量，函数返回后局部变量销毁）
2. **`add_done_callback(set.discard)` 自动清理**——防止 set 无限增长
3. **fire-and-forget 不返回 task** 给调用方（语义清晰）

## 测试 pattern

```python
@pytest.mark.asyncio
async def test_schedule_enrich_keeps_task_reference():
    from app.services.research.identity_enricher import schedule_enrich, _BG_TASKS

    initial = len(_BG_TASKS)

    # 触发后立即检查 set 有 +1
    schedule_enrich(user_id=1, company_name="Test Co")
    assert len(_BG_TASKS) == initial + 1

    # 等任务跑完，done_callback 自动 discard
    await asyncio.sleep(0.1)
    assert len(_BG_TASKS) == initial
```

## 实战案例：trade-ai-v1 M4-3

trade-ai-v1 在 M3 SoulReflector 用了反模式（`asyncio.create_task(_bg_reflect(...))` 不留引用），M4-3 IdentityEnricher 实施时被 code reviewer 抓到（对比 reviewer 提示原文："enricher 是冷路径，onboarding 后立刻 idle，丢任务概率比 reflect 高"），改成正模式。

修复后实测：onboarding 提交 → schedule_enrich 触发 → 主请求 200 立即返回 → 事件循环 idle → 后台 enricher 仍跑完写库（不丢任务）。

历史包袱：M3 的 `_bg_reflect` 仍是反模式，docstring 已点明可抄此 pattern，下次顺手改掉。

## 何时用

- ✅ Onboarding / 注册后异步爬数据
- ✅ 收到 webhook 后异步处理
- ✅ Soul / Memory 反思类后台任务
- ✅ 任何不需要返回值给调用方的后台事件

## 何时**不**用

- ❌ 需要等待结果的请求（直接 await）
- ❌ 任务时长 > 30 秒（请用真正的任务队列：Celery / Arq / RQ）
- ❌ 需要重试 / 持久化 / 跨进程（同上，task queue 不可省）

## 相关

- Python 3.11+ asyncio.create_task 文档：https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task
- trade-ai-v1 M4-3 PR #146 修复点：`server/app/services/research/identity_enricher.py`
