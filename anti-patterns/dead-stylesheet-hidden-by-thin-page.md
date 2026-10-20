# 反模式：整份样式表失效，却因为页面上没东西而藏了半年

## 一句话结论

一个页面的样式表**整份从未生效**，但谁也没发现 —— 因为那页当时只渲染一行占位文字，而那行恰好只用到了没被破坏的几个属性。**缺陷的可见性取决于页面内容量，不取决于有没有人去看。** 等内容补上去的那天，它一次性全部暴露，并且看起来像是"新改动把样式改坏了"。

## 场景

- 页面内容走运行时接口 / 功能未上线 / 数据为空，长期只渲染"暂无数据"之类的一行占位
- 预处理器（Less/Sass）或 CSS Modules 的配置与写法不匹配，产出非法 CSS
- 有人终于把内容填进去，页面当场难看 —— 第一嫌疑人必然是刚才那次改动

## 详细说明

### 踩坑经历

2026-10-20，给官网 `/faq` 做可抓取化：该页原先向接口取数据、爬虫只看到一句「暂无常见问题」，这次改成站内维护 14 条问答直接进预渲染产物。发到预览站，用户打开一看：文字顶满整个浏览器窗口、没有标题区、像"CSS 没加载"。

直觉结论是"刚才那次重写把样式改坏了"。实测否定了它：

```js
// 视口 1600，html font-size 160px
getComputedStyle(document.querySelector('.faq-content')).maxWidth  // "none"，样式表里写的是 1200
getComputedStyle(document.querySelector('.faq-header h1')).fontSize // "14px"，样式表里写的是 36
getComputedStyle(document.querySelector('.ant-collapse-header')).fontSize // "16px" —— antd 默认值
```

全是"样式表里有、页面上没有"。去编译产物里看，真相在那儿：

```
build/static/css/77.*.chunk.css
  .faq-content{margin:40rem/192 auto 0;max-width:1200rem/192;padding:0 20rem/192}   ← 没求值
对照 build/static/css/183.*.chunk.css（同项目另一个页面）
  .pricing-plans{background:#f8fafb;padding:.41667rem 1.19792rem .46875rem}          ← 正常
```

**两个独立死因叠加**，各自单独就足以致命：

1. 值写成 `36rem / 192` 这种手工除法。构建走 craco-less，Less 4 默认 `math: parens-division` —— **括号外的 `/` 不是除法**，原样输出 `font-size:36rem/192`，非法值，整条声明被浏览器丢弃。
2. 外面包了一层 `:global { }`。那是 CSS Modules 语法，而本项目不开 CSS Modules，`:global` 被原样输出进选择器，`.faq-content :global .ant-collapse` 成了非法伪类，整块规则失效。

只有**不含除法、且不在 `:global` 里**的属性活了下来：`background: linear-gradient(...)`、`text-align: center`、`color`、`font-weight`。所以它不是"CSS 没加载"，是"CSS 加载了但大部分声明被逐条丢弃"——这两者在肉眼下很像，在 `getComputedStyle` 下一眼可辨。

### 为什么能藏这么久

git 追溯显示：这些行在**半年前就是这样**，期间该页上过线、被预渲染抓过、被人打开过。它之所以没被发现，只有一个原因 —— **那时页面上只有一行「暂无常见问题」**，而这一行用的 `text-align: center; color: #999` 恰好不含除法、也不在 `:global` 里，看上去完全正常。

同一个文件里还有第三条死规则：答案区的样式写的是 antd 5 的类名 `.ant-collapse-content-box`，而产物里是 `.ant-collapse-body`。**改名也好、不改也好，它从来没生效过，因为外面还套着 `:global`。** 三条死规则在一个文件里共存半年，零症状。

这第三条值得单独记一笔：**覆盖组件库的内部类名，等于依赖一份没有版本号的契约**。antd 5→6 把 `ant-collapse-content-box` 改成 `ant-collapse-body`，升级日志里不会把它当 breaking change，而你的 less 规则只是静静地不再命中——没有构建错误、没有运行时报错、没有告警。上面第 1 条里的第三条 grep 就是针对它的：把 less 里出现的 `.ant-*` 类名逐个回产物里查，0 命中的即死规则。组件库大版本升级后应当跑一遍。

### 怎么防

1. **把编译产物当验收对象，而不是源码。** 一条 grep 就能拦住这一整类，不需要浏览器：

   ```bash
   # 未被求值的预处理器表达式泄漏进产物（容忍 `/` 两侧的空格，Less 原样输出时可能保留）
   grep -o "[0-9.]*rem */ *[0-9]*" build/static/css/*.css | wc -l   # 必须是 0
   # CSS Modules 语法泄漏进非 Modules 产物
   grep -c ":global" build/static/css/*.css                          # 必须是 0
   # 覆盖组件库内部类名的规则，逐个回产物里查有没有命中
   find src -name '*.less' -exec grep -oh "\.ant-[a-z0-9-]*" {} + | sort -u | while read c; do
     n=$(grep -oh -- "$c" build/static/css/*.css | wc -l | tr -d " ")
     [ "$n" = 0 ] && echo "未进产物: $c"
   done
   ```

   第三条的输出要会读。它判定的只是"**这条规则没出现在 shipped CSS 里**"，而这有两个原因，一个是缺陷、一个不是：

   - 上游改了类名、或选择器被破坏 → **真死规则**
   - 那个 less 文件根本没被引入（视图未被路由、组件已废弃）→ **误报**

   实测本仓：命令吐出 10 条，**10 条全是误报**，全部来自 `company-list` 与 `company-detail` 两个视图——它们在 `src/router/index.js` 里的路由是注释掉的。所以用法是**先按文件归并、排除未被引用的视图，再看剩下的**。另外它是粗筛：`.ant-collapse` 会被 `.ant-collapse-body` 命中，所以"有命中"不等于"你那条规则生效了"，只有"0 命中"是确定信号。

   本例实测：全站 12 处未求值除法，**全部集中在一个 chunk 里** —— 横向对比立刻指明了是哪一页的问题，而不是构建配置的通病。

2. **横向对比同项目的兄弟页面。** "我这页坏了"和"全站都这样"是完全不同的两个结论，而产物 CSS 的一次 grep 就能区分。本例正是靠 `pricing` 的产物正常，才把范围从"构建配置"缩到"这个文件的写法"。

3. **别信文件里的写法，去查项目的真实约定。** 同项目 `pricing/style.less` 第 1 行注释写着"桌面按 1920 设计稿写 px，postcss-pxtorem 自动转 rem"。FAQ 那个文件自己发明了一套 `/192` 手写除法 —— 没人验证过它在当前构建链下成不成立。

4. **内容为空的页面不算验证过。** 占位态能渲染正常，不能证明样式表是活的。要么用假数据灌满再看，要么用上面的产物断言。

## 数据支撑

| 验收口径 | 修复前 | 修复后 |
|---|---|---|
| `.faq-content` 计算 `max-width` | `none`（顶满 1600 视口） | `1000px`，左边距 300 居中 |
| `h1` 计算字号 | 14px | 30px |
| 答案区计算字号 | 16px（antd 默认，规则未命中） | 11.67px（设计 14px × 1600/1920） |
| 产物中未求值除法 | 12 处 | 0 |
| 产物中 `:global` | 存在 | 0 |
| 潜伏时长 | 至少半年，跨多次上线 | — |

## 适用范围

- **适用于**：任何「页面内容来自接口/功能未上线」而样式表早已写好的组合；预处理器或 CSS Modules 配置与写法可能不匹配的项目（Less 4 的 `parens-division`、Sass 的 `math.div`、CSS Modules 的 `:global`）
- **推广形态**：凡是"失败不报错、只是结果为零"的技术栈，缺陷的发现时间都由**有多少内容去触发它**决定，而不是由检查频率决定
- **不适用于**：会在构建期报错的语言/工具链（那类问题当场就暴露了）

## 相关

- [undefined-css-var-kills-declaration.md](undefined-css-var-kills-declaration.md) —— 同一个丢弃机制（非法值 → 整条声明作废、不报错不回退）。区别在于那条的盲区是 diff 评审看不到其他消费者，本条的盲区是页面上没有足够内容去触发规则。
- [visual-artifact-verified-by-reading-not-rendering.md](visual-artifact-verified-by-reading-not-rendering.md) —— 同样是整份样式表死掉。区别在于那条是**从没真渲染过**，而本条**渲染过无数次**（人看过、预渲染抓过）依然没发现。
- [dual-track-responsive-specificity-mismatch.md](dual-track-responsive-specificity-mismatch.md) —— 本条的直接后果：把这些死规则修活之后，桌面规则开始赢移动端规则，移动端字号被压到 3.12px。**修活一条长期失效的规则是一次隐性 breaking change。**
- [lazy-rendered-panels-missing-from-prerender.md](lazy-rendered-panels-missing-from-prerender.md) —— 两个缺陷互相掩护：答案区从来没显示过，所以答案区的死样式也一直没人发现。
- [seo-verified-in-build-never-in-production.md](seo-verified-in-build-never-in-production.md) —— 同一个官网项目更早的一条：只验本地 build、不验线上态。
