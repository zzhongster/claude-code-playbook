# 反模式：用 JS 合成事件驱动前端组件——它在一半交互上有效，正好骗你在另一半上找错方向

## 一句话结论

`el.click()` 和 `dispatchEvent(new MouseEvent(...))` 能点动简单按钮，于是很容易被当成通用手段。但**悬停展开、级联菜单、拖拽这类依赖真实指针轨迹的交互吃不到它**，而且失败是静默的（不报错、DOM 不变）。事件保真度是有阶梯的——**JS 合成 < 自动化工具的注入输入（CDP `Input.dispatchMouseEvent`）< 真机手势**——需要轨迹的交互直接从第二级起步，别在第一级上试错。

## 场景

用 CDP / Playwright 的 `evaluate` 驱动页面时，为了省事在页面里直接 `document.querySelector(...).click()`，或手工 `dispatchEvent` 造一串 `mouseover`/`mouseenter`。常见于：

- 目标元素没有稳定选择器、懒得算坐标
- 元素在滚动容器外，觉得"合成事件不受可见性限制"更方便
- 上一个按钮用 `.click()` 点开了，顺手复用同一套手法

## 踩坑经历（2026-09-04，trade-agent-data / WorkBuddy 连接器发布页）

一个两列级联下拉（左列一级分类、右列二级项）：

| 动作 | 手段 | 结果 |
|---|---|---|
| 展开顶层下拉 | `button.click()` | ✅ 生效——**正是这一下制造了"click 可用"的错觉** |
| 切换左列一级分类（右列跟着换） | `button.click()` | ❌ 右列纹丝不动 |
| 同上 | `scrollIntoView()` + 手工派发 `pointerover`/`mouseover`/`mousemove`/`mouseenter` | ❌ 仍然不动 |
| 同上 | CDP 真实鼠标（proxy 的 `/clickAt`，内部走 `Input.dispatchMouseEvent`） | ✅ 一次成功，右列立刻切换 |
| 选中右列二级项 | 同上，真实鼠标 | ✅ 值写回，菜单自动收起 |

组件为什么不吃合成事件没有深究（React 事件委托对 `isTrusted:false` 的 `click` 一般是认的，所以更可能是它的展开逻辑挂在指针序列或 hover 状态机上，单发合成事件构不成序列）。**但判据不依赖机制**：真实输入一次就过，合成事件两轮都不过。

这次还叠了另一个坑——三次"点击一级分类"其实因为调用拼法错误压根没执行（见[相关](discarded-probe-output-reads-noop-as-behavior.md)），于是"合成事件不行"这个结论一开始是**用错误证据得出的正确结论**。通路修好后重测，`.click()` 确实无效，`/clickAt` 确实有效。

## 根因

**保真度阶梯，越往上越接近真实用户，代价也越高：**

| 层级 | 手段 | 拿得到什么 | 拿不到什么 |
|---|---|---|---|
| 1 · JS 合成 | `el.click()`、`dispatchEvent` | 简单 `onClick`；不受可见性/遮挡限制 | 指针轨迹与 hover 状态机、原生手势合成（dblclick/拖拽）、`isTrusted` 门槛的 API |
| 2 · 自动化注入 | CDP `Input.dispatchMouseEvent`、Playwright `page.mouse` | 真实指针序列与 hover、点击可信 | 部分原生手势合成（**双击在 CDP 下也可能不产生 `dblclick`**）、真机触摸语义 |
| 3 · 真机 | 人手 / 真实设备 | 全部 | —— |

另外两点让它更难被发现：

1. **失败是静默的**：合成事件不抛异常、不改 DOM，观察结果与"我的调用没送达""这个元素不是可点的那个"完全同形；
2. **层级 1 在简单场景下有效**，所以第一次成功会把它固化成默认手段——直到撞上级联/hover 才失效，而那时人更容易怀疑选择器或组件，不怀疑手段。

## 修复 / 正确做法

**1. 按交互类型选层级，别按顺手程度选。**

- 普通按钮、链接、表单控件赋值 → 层级 1 够用（`form_input` 之类的封装更稳）
- **悬停展开、级联菜单、tooltip、拖拽、滑块、右键菜单 → 直接层级 2**，不要先试 `.click()`
- 依赖原生手势合成（双击、长按、捏合）→ 层级 2 也未必够，准备好在应用侧留非手势入口（按钮）或自己按 `pointerup` 判定

**2. 层级 2 需要选择器或坐标，提前把它算出来。**
自定义组件常常没有 id/data 属性，用结构位置生成并**当场验证唯一性**：

```js
var i = [...left.children].findIndex(b => b.textContent.trim() === "商业服务");
var sel = "div.flex.max-h-80 > div:nth-child(1) > *:nth-child(" + (i+1) + ")";
document.querySelectorAll(sel).length === 1 && document.querySelector(sel).textContent  // 先确认命中且唯一
```

**3. 一次失败就升级，别在同一层级里堆花样。**
`.click()` 不动 → 直接上真实输入。手工补派发 `mouseover`+`mouseenter`+`pointerover` 是典型的原地打转：它们仍然是层级 1，凑再多也跨不过那道坎。

**4. 每一步都验状态，别连点。**
级联菜单每点一次结构就变（列数、索引、菜单是否关闭）。点完立刻回读一次"当前选中值 + 当前列内容"，用它决定下一步的选择器——比一次性排好三步再执行可靠得多。

**5. 操作别人的表单时，先确认自己在改哪一格。**
本例下拉是复用组件：给第 2 项做的操作，如果第 1 项的菜单还开着，就会写进第 1 项。动手前先 Esc 关掉上一个菜单并回读所有槽位的值。

## 反模式信号

- 同一个 `.click()` 在 A 处成功、B 处失败，然后开始怀疑 B 的选择器
- 开始手工 `dispatchEvent` 拼鼠标事件序列——这是在层级 1 里模拟层级 2，几乎总是白费
- 对着"hover 才展开"的菜单反复调 `.click()`
- 结论里出现"这个组件大概需要 hover"，但从没用真实输入验证过

## 适用范围

- **适用于**：CDP / Playwright / Puppeteer / Selenium 驱动的真实前端（React、Vue、Angular 及其组件库都一样）
- **不适用于**：纯静态页面、自己写的裸 DOM 事件监听（`addEventListener("click")` 吃合成事件没问题）
- **注意**：层级 2 也有天花板，见下方相关条目里"CDP 注入不产生原生 dblclick"那条

## 相关

- [丢掉动作的返回值，把"没执行"读成"被测对象的行为"](discarded-probe-output-reads-noop-as-behavior.md)：同一次会话的上半场，两个坑叠在一起会让人对组件建出完全错误的模型
- [把浏览器自动化环境的假象当应用 bug](browser-automation-env-false-negatives.md)：阶梯更上一级的失效（CDP 注入拿不到原生手势），以及后台标签页 rAF/定时器节流那批环境假象
- 项目源：`trade-agent-data`，会话归档 `docs/sessions/0904-yc35.md`
