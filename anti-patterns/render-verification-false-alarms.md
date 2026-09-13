# 反模式：拿无头截图当手机验收结论——渲染工具自己的怪癖会报假警

## 一句话结论

"视觉产物必须真渲染"是对的，但**渲染工具本身会撒谎**：无头 Chrome 设 `--window-size=390,…` 实际按约 500px 布局再裁成 390 宽，页面看起来右侧被截断；带 `#锚点` 的 URL 截出来整片空白；浏览器面板的移动端模拟在页面加载之后才套用，首帧初始化逻辑读到的是桌面宽度。一次 HTML 报告验收里三次差点把工具怪癖当页面 bug 去"修"。**布局类问题用 DOM 量值判（`scrollWidth` vs `innerWidth`、元素 `getBoundingClientRect`），截图只用来看视觉。**

## 场景

- 本地生成单文件 HTML 报告/图鉴，没有 GUI 浏览器，用 headless Chrome 截图验收
- 需要验移动端宽度、长页面下半部分、依赖视口宽度的初始化逻辑（如"窄屏时图表横向滚动并停在右端"）

## 详细说明

| 现象 | 实际原因 | 怎么判别 | 替代做法 |
|---|---|---|---|
| 390 宽截图里整页右侧被截断，像横向溢出 | 桌面版无头 Chrome 有最小窗口宽度（约 500px），按 500 布局、按 390 裁图 | 在真实移动视口里执行 `document.documentElement.scrollWidth` 与 `innerWidth`，结果 375 / 375，并列出越界元素为空 | 手机验收用浏览器设备模拟 + DOM 量值，不看无头窄截图 |
| `file://…/page.html#s11` 截图全空白 | 跳到锚点后无头截图不绘制滚动区域 | 同一页面从顶部截是正常的 | 生成调试副本，注入 CSS 隐藏前面的章节，从顶部截 |
| 移动端模拟下"折线图滚到最右"没生效，`scrollLeft` 为 0 | 视口模拟晚于首帧：首帧容器未溢出，`requestAnimationFrame` 里的贴右是空操作；之后视口变化又复位滚动 | 手动设 `scrollLeft` 立即生效（0 → 323）；在一开始就是窄布局的环境里加载（无头 500px 宽）读到 198 / 198 | 初始化逻辑用 `ResizeObserver` 兜住尺寸变化，真机旋转屏幕也受益；验证放在"从一开始就是窄布局"的环境 |
| 程序滚动后立刻截图空白 | 截图早于重绘 | 等待后再截即正常 | `scrollTo` 之后 await 约 400ms 再截图 |

### 探针技巧：让 `--dump-dom` 带出运行时量值

`--dump-dom` 只输出 DOM，拿不到滚动位置和尺寸。给调试副本注入一段脚本，把量值写进标题：

```html
<script>setTimeout(function(){
  var p=document.querySelector('#fig-rel .plot');
  document.title='VW'+innerWidth+' sl='+p.scrollLeft+'/'+(p.scrollWidth-p.clientWidth);
},2000)</script>
```

然后 `chrome --headless=new --window-size=400,1600 --virtual-time-budget=6000 --dump-dom file://… | grep -o '<title>[^<]*</title>'`，标题里就是量值。

## 数据支撑

- 一次验收中出现 4 类假象，均用 DOM 量值或换截图方式排除
- 若按截图"修"第一个假象（给页面加更多窄屏收缩规则），会改坏真实手机上本来正确的布局

## 适用范围

- 适用于：无头浏览器截图验收 HTML/图表/报告页；带视口相关初始化逻辑的页面
- 不适用于：纯视觉风格检查（配色、字体、间距）——截图依然是最直接的

## 相关

- [AI 生成的视觉产物只读代码验收、从不真渲染](visual-artifact-verified-by-reading-not-rendering.md)——先要渲染；本条补充"渲染结果本身也要交叉验证"
- [浏览器自动化环境里的假阴性](browser-automation-env-false-negatives.md)
- [自写 md→docx 转换器四类静默渲染错](md-to-docx-converter-silent-rendering-bugs.md)
