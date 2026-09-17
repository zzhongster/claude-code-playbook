# 设计 token 换值的「只改颜色、不改布局」取证：同一份 DOM 注入旧值，逐元素比盒子

> 来源：TOUCH 2.0 界面改版 C1a（Next.js + Tailwind，颜色 token 以 RGB 通道存成 CSS 变量），把亮暗两组共 85 个颜色值整组换代，卡面完成判据要求「全站截图对比中只有颜色变化、无布局变化」——2026-09-18

## 一句话结论

证明一次 token 换值「只改颜色、不改布局」，别拿新旧两个分支各截一张图肉眼对。更干净的做法是：**在同一份 DOM 上**，先用新值渲染、记下每个元素的 `getBoundingClientRect`，再注入一段只含旧颜色值的样式、再记一次，逐元素比。两次只有 CSS 变量不同，结构、数据、字体、时序全部相同，差异只可能来自换值本身。前提是先把「页面自己会变」的来源全部冻住，并且证明注入确实生效；否则比出来的 0 可能是在拿同一份渲染自己跟自己比。

## 场景

- 设计系统换代：颜色、阴影等 token 整批换值，要在合并前给出「布局没被波及」的证据；
- token 走 CSS 变量（`--color-*`），Tailwind / 组件只引用变量名；
- 本地只能起部分页面（登录后的页面要库），但公开页和组件走查页能起。

两个分支各截一张图的做法有三个毛病：数据、时间、字体加载顺序不同，天然有像素差；肉眼对不出几像素的位移；对不出一千多个元素里的一个。

## 详细说明

### 1) 核心：同 DOM，两次测量

```js
// 旧 CSS 只取颜色变量，非颜色 token（圆角、阴影等）本次没改，不注入
const pick = (re) => [...(oldCss.match(re)?.[1] ?? "").matchAll(/(--color-[a-z0-9-]+):\s*([\d ]+);/g)]
  .map((m) => `${m[1]}: ${m[2]};`).join("\n");
// 特异度要压过页面自己的暗色媒体块 `:root:not(.light)`（0,2,0），所以用 (0,3,0)，暗色块放后面
const OLD_STYLE = `:root:root:root { ${pick(/:root \{([\s\S]*?)\n {2}\}/)} }\n` +
                  `:root:root.dark { ${pick(/\.dark \{([\s\S]*?)\n {2}\}/)} }`;

const rects = () => [...document.querySelectorAll("body *")]
  .filter((el) => !el.closest("nextjs-portal, script, style"))   // 排除 dev 浮层
  .map((el, i) => { const r = el.getBoundingClientRect();
    return `${i}:${el.tagName}:${Math.round(r.x)},${Math.round(r.y)},${Math.round(r.width)},${Math.round(r.height)}`; });

const page = await ctx.newPage();
await page.clock.install();                                  // ① 冻结计时器
await page.goto(url, { waitUntil: "load" });
await page.clock.pauseAt(new Date(Date.now() + 2000));       // 让首屏定时器跑完后停住
await ensureTheme(page, theme);                              // ② 走页面自己的主题机制并核实
await page.addStyleTag({ content:                            // ③ 停掉 CSS 动画（含带 !important 的）
  "*,*::before,*::after{animation:none!important;transition:none!important} :root .motion-spinner{animation:none!important}" });
const after = await page.evaluate(rects);
const again = await page.evaluate(rects);                    // ④ 自比：必须为 0，否则比较无效
await page.addStyleTag({ content: OLD_STYLE });
const before = await page.evaluate(rects);
const probe = await page.evaluate(() =>                      // ⑤ 探针：证明旧值真的注入进去了
  getComputedStyle(document.documentElement).getPropertyValue("--color-surface").trim());
```

报告里每组写三个数：元素数、自比差异数、新旧差异数，再加探针读数。

### 2) 让「0 差异」可信的五件事（缺一件，0 就可能是假的）

| # | 不做会怎样 | 做法 |
|---|---|---|
| ① 冻结计时器 | 组件走查页有定时演示（加载转圈、流式插行），两次测量之间 DOM 自己多了 1 个元素 | Playwright `page.clock.install()` + `pauseAt` |
| ② 主题用页面自己的机制并核实 | 走查页挂载时调 `applyThemeClass("light")`，覆盖了脚本先加的 `.dark`，「暗色」那组比的其实是亮色，截图也是亮色 | SSR 页种主题 cookie；走查页点它自己的切换开关；轮询核实 `<html>` 类，没切到就抛错 |
| ③ 停掉动画 | 旋转中的 svg 外接盒每帧都变，自比出现 9–14 处差异；`* { animation:none !important }` 压不住带 `!important` 的 `.motion-spinner`（0,1,0） | 通配 + 更高特异度的点名规则 |
| ④ 先做自比 | 上面三类问题都是靠自比暴露的，不做就直接拿到一个混了噪声的新旧比较 | 注入前连测两次，必须为 0 |
| ⑤ 探针变量选亮暗两侧新旧都不同的 | 注入被 `:root:not(.light)`（0,2,0）压住时，比较恒为 0；如果探针选的是 `--color-brand`（旧版暗色与亮色同值），读出来分不清是注入生效还是失效 | 选 `--color-surface` 这类亮暗新旧四值都不同的变量，报告里写读数：旧亮 `255 255 255`、旧暗 `15 23 42` |

### 3) 截图与测量分开做

冻结时钟后 `page.screenshot` 会一直等下一帧，30 秒超时。所以测量页冻结时钟，截图另开一页、不冻结。截图用元素级 `locator.screenshot`；走查页的 sticky 顶栏会在元素截图滚动时盖进区块，截图页里把它改成 `position: static`（只影响截图，不参与测量）。改版前后左右拼成一张图贴卡面。

## 数据支撑

- 3 个页面（组件走查页、落地页、定价页）× 亮暗，约 3,700 个元素（走查页 1,701 个），新旧色值盒子不一致 **0**，自比 **0**，探针读数均为旧值。
- 修正前依次踩到的假象：
  - 暗色实际是亮色：比较恒为 0，看截图才发现；
  - 注入旧暗色值被媒体块压住：探针读到新值；
  - 定时演示：元素数差 1；
  - 转圈动画：自比 9–14 处差异。

  每一个都会让结论失真，而其中两个的失真方向是「更绿」。

## 适用范围

- 适用于：颜色、阴影、透明度等「理论上不影响几何」的 token 换值；token 走 CSS 变量、能在运行时覆盖。
- 不适用于：字号、行高、间距、圆角、字体族换代，这些本来就会改几何，要用视觉对照而不是「0 差异」。也不适用于 token 在构建期被烤进类名的方案（运行时注入不了旧值）。
- 局限：只能覆盖本地起得来的页面。登录后页面要靠「共用同一份 token CSS、本次不改任何类名」这条推理，报告里要写明射程。
- 目前只在一个项目里验证过。

## 相关

- [反模式：拿无头截图当手机验收结论](../anti-patterns/render-verification-false-alarms.md)：同一个判断方向，布局用 DOM 量值判，截图只用来看视觉。
- [反模式：视觉产物只读代码验收](../anti-patterns/visual-artifact-verified-by-reading-not-rendering.md)
- [浏览器内无副作用验证](in-browser-side-effect-free-verification.md)：同样是在页面里临时注入、测完即走。
