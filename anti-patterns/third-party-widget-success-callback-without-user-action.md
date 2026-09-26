# 反模式：在第三方验证组件的 success 回调里直接提交表单——SDK 降级时会自己调用 success，页面变成无限自提交

## 一句话结论

人机验证、支付、登录这类第三方前端 SDK 的 `success` 回调，**不等于「用户刚完成了一次交互」**。SDK 初始化失败或服务不可达时，它的降级路径可能在**页面一加载时就自己调用 success**，并给你一个交给服务端判定的参数。回调里如果直接 `form.submit()`，服务端重新渲染的还是同一个页面，页面又加载同一个 SDK、又被自动调用 success，于是页面无限循环地自己提交。**只有真正由用户点击触发的那一次 success 才允许提交。**

## 场景

- 页面是服务端渲染的表单，第三方 SDK 只负责弹出验证，通过后由回调带着参数提交表单。
- 服务端对失败的处理是「原地重新渲染同一个页面，并显示错误」（这本身是好的 UX 做法）。
- SDK 的配置由环境变量注入：配错了或服务不可用时，SDK 进入降级模式。

## 踩坑经历

trade-agent-data 的 OAuth 授权页（2026-09-26），接入的是阿里云验证码 2.0：

```js
window.initAliyunCaptcha({
  SceneId: "…", mode: "popup", button: "#captcha-button",
  success: function (param) {
    appendHidden("captcha_param", param);
    appendHidden("action", "send");
    form.submit();
  },
});
```

用本地假配置（`prefix: "pfx1"`）渲染页面后在浏览器里打开：**不点任何按钮**，页面加载约一秒后就 POST 到 `/oauth/phone`。在 `HTMLFormElement.prototype.submit` 上挂 hook 抓调用栈，结果是 `AliyunCaptcha.js → Y.success → form.submit`，同时控制台有一条 `Uncaught (in promise) Error: Network Error`，也就是初始化失败后走了降级路径。

换成真实的场景 ID 和身份标以后不再自动触发，所以**只在配置错误或服务故障时出现**，也就是正常测试永远碰不到的时候。服务端的处理是手机号为空 → 重新渲染并提示错误 → 页面重新加载 SDK → 再次自动调用 success……一路循环下去，直到撞上按 IP 的限流。

## 为什么难发现

- 单元测试不跑前端 SDK，服务端测试全部通过。
- 用正确配置人工测试，一切正常。
- 出问题的条件（SDK 降级）刚好就是线上故障的时刻，这时候没人盯着页面。

## 替代做法

1. **只有用户动作才能启动提交**：在按钮上挂一个 capture 阶段的 click 监听，只有 `e.isTrusted` 为真时才设 `armed = true`；success 回调里先读取再清零（`var ok = armed; armed = false; if (!ok) return;`），一次点击只能换来一次提交。
2. **`fail` 回调里也要清零**，否则一次失败的点击会留下 armed 状态，下一次 SDK 自动调用 success 时还能提交一次。
3. **提交前检查业务前置条件**（例如手机号不能为空），不满足就只提示、不提交。
4. **验证方式**：用一份故意写错的配置渲染页面，放进真实浏览器，hook 住 `submit` 看有没有调用。本地 `file://` 页面上浏览器工具不能用，要起一个本地 HTTP 服务来打开。

## 适用范围

- 适用于：人机验证、第三方登录、支付，以及任何「SDK 回调 → 自动提交或跳转」的集成，尤其是失败时会渲染回同一个页面的服务端渲染流程。
- 不适用于：回调只更新本地 UI、不触发网络请求的场景。

## 相关

- [synthetic-events-dont-drive-framework-components.md](synthetic-events-dont-drive-framework-components.md)
- [fake-dom-missing-sibling-controls-hides-name-collision.md](fake-dom-missing-sibling-controls-hides-name-collision.md)：这次修复本身又引入了一个 bug，见这一篇
