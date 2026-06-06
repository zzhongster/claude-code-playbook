# 叠加 Modal 的导航返回栈断链 — 子层关闭不回父层，整条流退出

## TL;DR

A modal 打开 B modal（或 drawer / 多步流）时，B 的 `onClose` **只关自己、没恢复 A** →
用户在 B 点「取消 / ✕ / ESC」后**整条流直接退出**，而不是退回 A。

这类 bug 阴险在于：**单测「B 能打开」永远通过**，漏的是「B 退出后的落点」。
DOM 断言（"B 出现了"）测不到导航栈断链。必须对**每个退出口**断言**落点**。

## 症状（trade-ai #316 实测）

实体锚定 modal（A）零候选时给「补线索」入口 → 打开补线索 modal（B）。
用户在 B 点「取消」→ 期望回到锚定页（A）继续选，实际**两个 modal 都没了，回到裸报告页**。

## 根因

父→子只做了「关父、开子」，子→父的回程没接：

```tsx
// 父层：点"补线索"
const handleAnchorClue = () => {
  setAnchorModalOpen(false);   // 关 A
  setClueModalOpen(true);      // 开 B
};

// ❌ 子层 onClose 只关自己 → A 永久丢失
<ClueInputModal onClose={() => setClueModalOpen(false)} />
```

导航栈是「A → B」，但 B 的退出只 pop 了 B，没 push 回 A。

## 修复

子层的「取消 / ✕ / ESC」回程要把父层推回来（父层 state 没被清，候选/空状态都还在）：

```tsx
// ✅ 取消子层 → 回父层
<ClueInputModal onClose={() => { setClueModalOpen(false); setAnchorModalOpen(true); }} />
```

注意区分两类退出：
- **取消类**（取消 / ✕ / ESC / 返回）→ 回父层
- **提交/完成** → 按业务走（可能回父层、可能前进到下一步、可能整体关闭）

## 通用测试方法：导航返回栈矩阵

对每个 `父 →(动作)→ 子` 的转换，枚举子层的**所有退出口**，逐个断言落点：

| 退出口 | 期望落点 |
|---|---|
| ✕（头部） | 回父层 |
| 取消（footer） | 回父层 |
| ESC | 回父层 |
| backdrop 点击（若允许） | 回父层 |
| 提交 / 完成 | 业务定义（前进 or 回父 or 关闭） |

红线：**任何取消类退出都不允许出现「父子都没了」的断链退出**。

这是黑盒 D-B-R-S-V 里 **S（状态机闭环）** 的高频子类——不只是「A→B→取消→回A」单条，
而是「**每个退出口 × 每个层级转换**」的矩阵。

## 可复用 helper（playwright sync）

```python
def assert_modal_backstack(page, *, enter, parent, child, exits, settle_ms=600):
    """每个退出口独立验证：enter(父→子) → 验父隐藏 → 触发退出 → 验落点。
    enter 必须幂等（每次都能从当前态打开子层，可含 mock）。
    exits: [{"name", "action": callable, "expect": "parent"|"closed"}]
    """
    def _cnt(t): return page.locator(f'[data-testid="{t}"]').count()
    for ex in exits:
        enter()
        page.wait_for_selector(f'[data-testid="{child}"]', timeout=8000)
        assert _cnt(parent) == 0, f"[{ex['name']}] 进子层后父层应隐藏"
        page.wait_for_timeout(settle_ms)
        ex["action"]()
        page.wait_for_timeout(settle_ms)
        child_gone, parent_back = _cnt(child) == 0, _cnt(parent) > 0
        if ex["expect"] == "parent":
            assert child_gone and parent_back, f"[{ex['name']}] 应回父层，实际断链退出"
        else:
            assert child_gone and not parent_back, f"[{ex['name']}] 应整体关闭"
        if parent_back:
            page.keyboard.press("Escape"); page.wait_for_timeout(200)  # 复位
```

用法（一行覆盖一个流的所有退出口）：

```python
assert_modal_backstack(page, enter=open_child, parent="anchor-modal", child="clue-modal",
    exits=[
        {"name":"✕",   "action": lambda: page.locator('[data-testid="clue-cancel"]').click(),        "expect":"parent"},
        {"name":"取消", "action": lambda: page.locator('[data-testid="clue-cancel-footer"]').click(), "expect":"parent"},
        {"name":"ESC",  "action": lambda: page.keyboard.press("Escape"),                              "expect":"parent"},
    ])
```

## 适用范围

任何**叠加式 / 多步 UI**：modal 套 modal、modal 开 drawer、向导式多步表单、抽屉里再开抽屉。
只要存在「进入下一层 → 可取消返回」的路径，就该跑导航返回栈矩阵。

前置：给每个 modal 根节点 + 每个退出口加 `data-testid`（测试可达性最低成本投资）。

## 关联

- 属于黑盒 D-B-R-S-V 的 **S（状态机闭环）**，但要测「退出口 × 层级」矩阵而非单条闭环
- 「DOM 存在 ≠ 功能正常」的又一例：B 能打开 ≠ 导航流正确
- 修复成本一行（onClose 推回父层），但漏测成本是整条交互流不可用
