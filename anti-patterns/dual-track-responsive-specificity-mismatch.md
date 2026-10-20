# 反模式：桌面/移动双轨样式靠特异性分胜负——修活一侧就压垮另一侧

## 一句话结论

**媒体查询不提升特异性。** 桌面块的选择器写了 4 层、移动端块只有 3 层，而移动端元素同时命中两条规则时**桌面值赢**——只要桌面那一侧长期是死的，这个错位就毫无症状。于是**把一条长期失效的规则修活，本身就是一次隐性 breaking change**：本例修完桌面，移动端的问题标题被压到 **3.12px**（几乎不可见）。双轨样式必须让两侧层级对齐、靠**源码顺序**取胜，不能靠多加一个类。

## 场景

- 同一组件两套样式：桌面一块 + 移动端一块（`@media (max-width: …)`，或另一个根类如 `.foo-nopx`）
- 移动端容器**同时**带两个根类（rem 转换、主题、栅格方案常这么干），两侧规则都命中它
- 其中一侧长期失效（规则是死的、类名被上游改过、文件没被引入），冲突从未发生过
- 有人终于把失效那一侧修活了

## 详细说明

### 机制

CSS 规则的胜负只看两件事：**特异性，然后源码顺序**。`@media` 包一层不参与特异性计算，也不构成更高优先级——它只是让规则在条件不满足时不参与。

```less
// 桌面块：4 个类 → (0,4,0)
.faq .faq-content .ant-collapse .ant-collapse-header { font-size: 16px; }

// 移动端块：3 个类 → (0,3,0)，写在文件后面，但赢不了
@media (max-width: 768px) {
  .faq-nopx .faq-content .ant-collapse-header { font-size: 15px; }
}
```

移动端的那个元素**同时带 `.faq` 和 `.faq-nopx`**，两条都命中，4 层 > 3 层，桌面值赢。媒体查询在后面也没用。

### 为什么以前没事，一修就炸

桌面那一侧整份是死的（Less 手工除法叠加 `:global`，见 [dead-stylesheet-hidden-by-thin-page.md](dead-stylesheet-hidden-by-thin-page.md)）。**规则不存在 = 冲突不存在。** 修活的那一刻，一条从未参与过布局的规则突然开始赢，而没有任何人把这次修复当成"会改变移动端"的改动。

**双轨 rem 方案会把这个错位放大成数量级错误。** 两轨的 root font-size 基准不同：桌面按 1920 设计稿，1rem ≈ 视口/10（1600 视口下 160px）；移动端按 375 设计稿，1rem = 37.5px。桌面规则换算出来是 `0.0833rem`，在桌面是 13.33px，漏到移动端就是 `0.0833 × 37.5 = 3.12px`——实测值与这个反推完全吻合。所以双轨泄漏不是"字号略有偏差"，是**差一个数量级**。

### 改法

把两侧层级拉平，胜负交给源码顺序（移动端块在后）：

```less
// 桌面：header/body 不再嵌进 .ant-collapse，拉平成 3 层
.faq .faq-content .ant-collapse-header { … }
.faq .faq-content .ant-collapse-body   { … }

// 移动端：同为 3 层，写在后面，靠顺序赢
@media (max-width: 768px) {
  .faq-nopx .faq-content .ant-collapse-header { … }
  .faq-nopx .faq-content .ant-collapse-body   { … }   // 写在 body 上，不写在它的 p 上，与桌面同级，p 继承
}
```

注意最后那句注释：**哪一层承载声明也要对齐。** 原来移动端把答案字号写在 `.ant-collapse-body p` 上，桌面写在 `.ant-collapse-body` 上——层数对齐了但承载元素不同，一样会错配。

## 怎么防

1. **双轨样式逐条对齐选择器层级**，胜负永远交给源码顺序。想赢就把块放在后面，不要靠多写一个祖先类。
2. **修一条长期失效的样式规则，按 breaking change 对待**：必须双视口回归，而不是"我只是把它改对了"。
3. **量计算值，不看源码**。改前改后在两个视口各量一遍，差异必须是你预期的那几项：

   ```js
   // 两个视口各跑一次，比对输出
   ['.faq-content','.faq-header h1','.ant-collapse-header','.ant-collapse-body']
     .map(s => [s, getComputedStyle(document.querySelector(s)).fontSize,
                   getComputedStyle(document.querySelector(s)).maxWidth]);
   ```

4. 怀疑双轨泄漏时，直接问"这个元素挂了几个根类"：`document.querySelector('.faq-content').closest('[class*=faq]').className`——同时出现两轨的根类就是高风险信号。

## 数据支撑

同一页、同一次修复的三个状态（视口 1600 / 375）：

| 项 | 桌面规则还是死的 | 修活桌面后 | 层级拉平后 |
|---|---|---|---|
| 桌面 问题标题 | 16px（antd 默认，规则未命中） | 13.33px ✅ | 13.33px ✅ |
| 桌面 `.faq-content` max-width | `none`（顶满 1600） | 1000px ✅ | 1000px ✅ |
| **移动端 问题标题** | 15px ✅ | **3.12px ❌** | 15px ✅ |

中间那一列是"修复提交如果直接发出去"的样子：桌面三项全部转好，移动端主标题变成不可读——**而验收只看桌面的话，这一列是全绿的。**

## 适用范围

- **适用于**：任何桌面/移动分块的样式（媒体查询双块、双根类、双 rem 基准、双主题）；尤其是其中一侧可能长期失效的项目
- **同类风险**：`@supports`、`:dir()`、容器查询——同样不提升特异性；靠"包一层条件"来表达优先级一律不成立
- **不适用于**：CSS Modules / scoped CSS 下各组件自带哈希、根本不共享选择器空间的工程（但同一组件内部的双块依然适用）

## 相关

- [整份样式表失效，却因为页面上没东西而藏了半年](dead-stylesheet-hidden-by-thin-page.md) —— 本条是它的直接后果：死规则一旦修活，就暴露了它活着时本来就有的冲突
- [懒渲染组件吃掉预渲染正文](lazy-rendered-panels-missing-from-prerender.md) —— 同一个页面、同一轮改造里的另一条
- [无头截图当手机验收结论](render-verification-false-alarms.md) —— 双视口回归要量 DOM 值，不要只看截图
