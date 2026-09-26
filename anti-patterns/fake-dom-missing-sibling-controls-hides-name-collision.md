# 反模式：假 DOM 只放被测元素，漏掉同名的兄弟控件——`form.elements` 同名解析和 `submit()` 不带按钮值这两个坑，测试都看不见

## 一句话结论

给前端脚本写 node 行为测试时，假的 `form` 如果只放了「脚本会用到的几个元素」，那么和**页面上其它控件**有关的 DOM 语义全都测不到。最典型的是：`form.elements["action"]` 会返回**同名的提交按钮**，而不是你以为会新建的隐藏字段；`form.submit()` **不会带上任何按钮的值**。两件事叠在一起，请求就少了一个关键参数，但单元测试是绿的。**假 DOM 的结构要照真实页面来搭，不能照脚本的需要来搭。**

## 场景

- 服务端渲染的表单里，有一个提交按钮用 `name="action" value="login"`，靠 `name/value` 区分动作（这是无 JS 表单常见的写法）。
- 另一条路径由 JS 触发（比如人机验证通过后），需要以另一个 `action` 值提交同一个表单。
- 为了避免重复添加字段，写了一个「有就复用、没有就新建」的辅助函数：

```js
function setHidden(name, value) {
  var el = form.elements[name];          // ← 找到的是 <button name="action">
  if (!el) { el = document.createElement("input"); el.type = "hidden"; el.name = name; form.appendChild(el); }
  el.value = value;                      // 改掉了按钮的 value, 没有新建隐藏字段
}
setHidden("action", "send");
form.submit();                           // submit() 不带按钮值 → 请求里根本没有 action
```

## 踩坑经历

trade-agent-data 授权页（2026-09-26）。为修上一个坑（见相关的第一篇），把「每次都新建隐藏字段」改成了上面这种「先复用再新建」。改完的结果：

- 写了一个 node 行为测试，把页面上的真实脚本抽出来跑，fake `form.elements` 里**只有 `phone`**，断言「点击之后 success 回调只提交一次」，测试通过。
- 全量 2978 个单测通过，逐任务评审也通过了。
- **最后的整分支终审**在真实浏览器里用 `new FormData(form)` 看提交数据，结果只有 `phone, code, captcha_param`，**没有 `action`**；而 `form.elements["action"].tagName === "BUTTON"`。服务端于是走进了「登录」分支，回复「请先获取验证码」。**线上短信一条都发不出去**，而手机号登录是默认 tab。

修复前的版本每次都 `createElement` 新建隐藏字段，反而是对的。这次「优化」和测试的缺口正好出现在同一个地方。

## 为什么测试会漏

- 假 DOM 是按脚本的需要搭的：脚本用了 `phone`，就只放 `phone`。
- 这个 bug 来自**脚本没有直接用到、但和它同名的控件**，而且只在两条浏览器语义同时成立时才出现：`elements[name]` 是按 name 查找的，`submit()` 不带提交者。
- 断言的是「提交了几次」，而不是「提交出去的数据是什么」。

## 替代做法

1. **假表单的控件照真实模板一个不少地放进去**，特别是同名按钮、同名 radio 这类会让 name 冲突的控件。假 DOM 也要照真实浏览器的规则来：`elements[name]` 按 name 查找，`submit()` 只收集可提交的 input，不收集按钮。
2. **断言提交出去的数据**，也就是模拟出的 `FormData`，而不是只数提交次数。
3. **查找隐藏字段要限定类型**，写成 `form.querySelector('input[type="hidden"][name="' + name + '"]')`，或者干脆给这个字段起一个不会和其它控件冲突的名字。
4. **先让测试在旧实现上变红**，确认这个测试真能抓到这个 bug，再去修（参见 tdd-fake-red）。

## 适用范围

- 适用于：用 node 或 jsdom 跑页面内联脚本的测试，只要表单里有按钮、radio 这类同名控件。
- 不适用于：直接在真实浏览器（Playwright 等）里渲染真实页面的 e2e 测试，那里的 DOM 语义天然是对的。

## 相关

- [third-party-widget-success-callback-without-user-action.md](third-party-widget-success-callback-without-user-action.md)：引出这次修改的上一个坑
- [tdd-fake-red.md](tdd-fake-red.md)
- [guard-passes-without-the-fix.md](guard-passes-without-the-fix.md)
