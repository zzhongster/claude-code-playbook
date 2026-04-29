# Vanilla JS 组件 listener 生命周期管理

> 写不依赖 React/Vue 的纯 JS 组件时，`addEventListener` 累积 + `innerHTML` 重渲染 = 经典内存泄漏陷阱。这份是工作流。

## 适用场景

- 用 vanilla JS class 写可复用前端组件（无框架）
- 组件需要 `innerHTML = '...'` 重渲染部分 DOM
- 组件有 `destroy()` API 让外层卸载
- 全局 listener（document/window keydown / beforeunload）跟组件交互

## 核心问题

```js
class MyComp {
    addRow() {
        const btn = document.createElement('button');
        btn.addEventListener('click', this.handler);
        this.container.appendChild(btn);
    }

    rerender() {
        this.container.innerHTML = '...';   // ❌ 旧 button GC，但 handler 闭包 + listener 引用都不会被清
    }
}
```

每次 `rerender()` 完，旧 button DOM 节点确实从 tree 上脱离，浏览器 GC 会回收节点。但是：
- 如果你把 listener 自己存到 `this._listeners` 数组里跟踪 → 数组累积 stale 引用
- 即使没存数组，旧 button 上的 listener 仍引用 `this.handler`，handler 引用 `this` 闭包 — 短期 GC 会清，但**长会话**（用户连续编辑 100+ 次）会让数组膨胀，destroy 时遍历开销大

## 推荐范式

### 1. 区分 listener 类型

| 类型 | 例子 | 何时清 |
|---|---|---|
| **Persistent**（一次性绑） | document keydown 全局快捷键、window beforeunload、容器外的 ⋯ menu | destroy 时清 |
| **Container-scoped**（每次重渲染绑） | container 内部的 button click、chip 删除 ✕ | **每次重渲染前**清 |

### 2. 跟踪 + 选择性清理

```js
class MyComp {
    constructor() {
        this._listeners = [];  // [{target, event, handler}]
    }

    _on(target, event, handler) {
        target.addEventListener(event, handler);
        this._listeners.push({ target, event, handler });
    }

    // 重渲染前调用：只清「容器内」的 listener
    _clearLocalListeners() {
        this._listeners = this._listeners.filter(({ target, event, handler }) => {
            if (this.container.contains(target)) {
                target.removeEventListener(event, handler);
                return false;
            }
            return true;  // 保留 document/window 级
        });
    }

    rerender() {
        this._clearLocalListeners();      // ← 关键
        this.container.innerHTML = '...';
        this._bindButtons();              // 重新绑
    }

    destroy() {
        // 清全部（含 document keydown / window beforeunload）
        this._listeners.forEach(({ target, event, handler }) => target.removeEventListener(event, handler));
        this._listeners = [];
        clearTimeout(this._someTimer);    // 别忘了 timer
        this.container.innerHTML = '';
    }
}
```

### 3. 注意子区域 vs 整个 container 的边界

如果组件是 head（一次性绑）+ body（频繁重绘），不能粗暴清整个 container。

错例（trade-ai #179 实战踩到的）：

```js
_render() {
    this.container.innerHTML = `<div class="head"><button data-act="reset">重置</button></div>
                                <div class="row"></div>`;
    this._on(reset, 'click', ...);   // head 区一次性绑
    this._renderRows();
}

_renderRows() {
    this._clearLocalListeners();      // ❌ 把 head 的 reset listener 也清了
    this.rowEl.innerHTML = '...';     // 只重绘 row 区
}
```

修复：精准定位「row 区内」的 listener：

```js
_clearRowListeners() {
    this._listeners = this._listeners.filter(({ target, event, handler }) => {
        if (this.rowEl.contains(target)) {
            target.removeEventListener(event, handler);
            return false;
        }
        return true;
    });
}
```

## 触发点 checklist

写新组件时，问自己：

- [ ] `_listeners` 数组有没有？
- [ ] 每个 `addEventListener` 都走 `_on()` 吗？
- [ ] 有 `destroy()` 吗？里面解绑全部 listener？
- [ ] 有 `setTimeout` / `setInterval`？destroy 里 `clearTimeout`？
- [ ] 有 document/window 级 listener？destroy 解绑了？
- [ ] 有 `innerHTML = '...'` 重渲染？前面调了 `_clearXxxListeners()`？
- [ ] 是否有 head 区一次性绑 + body 区频繁重绘？分两个清理函数？
- [ ] beforeunload 在 dirty 状态外是否被解绑？

## 反模式

### ❌ 全局 keydown 不区分焦点

```js
document.addEventListener('keydown', (e) => {
    if (e.key === 'e') this.enterEdit();   // 用户在 input 里输 e 也触发
});
```

修：

```js
document.addEventListener('keydown', (e) => {
    const tag = document.activeElement?.tagName || '';
    if (['INPUT', 'TEXTAREA', 'SELECT'].includes(tag) || document.activeElement?.isContentEditable) return;
    if (e.key === 'e') this.enterEdit();
});
```

### ❌ Modal 单例不清旧的

```js
let activeOverlay = null;

function open(opts) {
    if (activeOverlay) _detach();   // ❌ 直接 DOM 卸下，不触发 onClose 回调
    // ... new modal
}
```

修：

```js
function open(opts) {
    if (activeOverlay) close();     // ✅ 走 close 触发 onClose 通知业务方
    // ...
}
```

## 实战案例

trade-ai #179 M2 7 个组件，audit 累计找到 4 个相关 bug：
- MaterialDropZone destroy 不清 _noteTimer（4s 后 callback 抛 null）
- FieldConflictModal close 重复调用 + Esc keydown 不释放
- KPDetailDrawer 覆盖式 open 不触发 onClose
- KPFilterPanel `_clearLocalListeners` 误清 head 区 reset listener

每个问题在 audit 中暴露，加上面 checklist 后基本不漏。
