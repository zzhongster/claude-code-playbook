# 反模式：预渲染 + 懒加载路由 = 每一页 hydration 必败，而且没有任何测试会红

## 一句话结论

全站路由走 `React.lazy` + `<Suspense fallback>` 时，预渲染出来的**每一页**在客户端首帧渲染的都是 fallback（chunk 还没下载完），与静态 HTML 不符 → React 报 **#418**（hydration 文本不匹配）→ **#423**（放弃 hydration，整个 root 改走客户端渲染）。对爬虫零影响（静态 HTML 是完整的），但**把 hydration 的全部性能收益吃掉了**。它不报业务错、不白屏超过一瞬、不影响任何断言，所以能长期存在且每个改动者都会怀疑是自己刚引入的。判定"是不是我改出来的"有确定手段：**用改动前的 commit 另建一份对照产物，在未改动的页面上比报错。**

## 场景

- CRA / Vite 的 SPA + react-snap 之类的预渲染，React 18 的 `hydrateRoot`
- 路由为了拆包全部写成 `React.lazy(() => import('./views/X'))`，外面套一个全局 `<Suspense fallback={<Loading />}>`
- 打开任意预渲染页，控制台两条 minified React error，页面功能完全正常

## 详细说明

### 机制（有两个候选根因，都未用非压缩 React 坐实）

**候选一（本次对照实验指向的）**：预渲染产物里是完整页面；客户端 `hydrateRoot` 的第一帧，lazy chunk 尚未加载，`Suspense` 渲染的是 `<Loading />`。首帧 DOM 与 HTML 不匹配 → #418 → React 判定无法协调 → #423 整树重渲染。

**候选二**：同项目 wiki 2026-07-29 把这个全站现象记成 **AntD 6 cssinjs 的样式注入顺序**问题。

两者都能解释"全站每一页都报、与页面内容无关"。**本条没有用非压缩版 React 验证过**，#418/#423 只给编号不给具体不匹配的节点；要定案需要 `react-dom/profiling` 或开发构建下看完整报错。写进 playbook 的是**现象与判定方法**，不是已证实的根因。

### 影响评估（别修反了）

| 关心的事 | 受影响吗 |
|---|---|
| 爬虫 / SEO / GEO | **不受影响**——静态 HTML 是完整的，爬虫不执行 hydration |
| 结构化数据、meta、canonical | 直接加载时全部正确（SPA 站内跳转后 `<head>` 不完全更新，但爬虫看到的是直接加载那一种） |
| 首屏性能 | **受影响**——首屏闪一下 `Loading`，然后整页客户端重渲染；hydration 省下的那次渲染全还回去了 |
| 功能正确性 | 不受影响，内容最终一致 |

所以它的优先级是"性能债"，不是"线上故障"。**真要修是全站级改动**：路由从 lazy 改同步（牺牲拆包）、或给 react-snap 配 chunk preload 让首帧就有组件、或改用支持流式 hydration 的框架。不要在一个内容页的改动里顺手动它。

### 关键手段：对照产物法

这类"全站既有缺陷"最大的成本不是修它，是**每个人都要重新怀疑一次自己**。把怀疑变成判定：

```bash
# 1) 用改动前的 commit 另建一份产物（worktree 不污染当前工作区）
git worktree add /tmp/ctrl <改动前的 commit>
cd /tmp/ctrl && npm ci && npm run build

# 2) 在「这次完全没碰过」的同一个页面上，比两份产物的控制台报错：种类与条数
#    （用不带 -s 的 serve 起静态服务，-s 会把子路径重写到首页）
```

判定规则：

- 未改动的页面在两份产物里报**同样的错、同样的条数** → 既有缺陷，不是你引入的，照常交付并记档
- 只有你改的页面报 → 你引入的
- 对照产物里没有、你的产物里全站都有 → 你改到了全站级的东西（入口、模板、构建配置）

本次就是靠这个把"要不要回滚我的改动"从猜变成判：取 HEAD 另建对照产物，未改动的 `/pricing` 上报的是同样两条、同样数量。

## 怎么防

1. **预渲染项目把 hydration 报错纳入基线**：在仓里记一条"当前全站 #418+#423，根因候选 X/Y，影响仅性能"，省掉后面每个人的重新排查。
2. **怀疑自己之前先建对照产物**。比"读代码推断"快，而且结论是二值的。
3. 真要定案根因，用开发构建（非 minified React）跑一次，报错会指出不匹配的具体节点。
4. 验收预渲染页时把"控制台报错条数"和"静态 HTML 内容完整性"**分成两条验收项**——前者可能长期是红的，不该挡住后者。

## 数据支撑

- `/faq`、`/about`、`/privacy`、`/pricing` 四页**全部**报 #418 + #423；其中 `/about`、`/privacy`、`/pricing` 本次一字未改
- 对照产物（HEAD `e68cff6` 另建）在未改动的 `/pricing` 上报**同样两条、同样数量**
- 连带现象：SPA 站内跳转 `/faq → /about` 后 `title` 更新、`canonical` 仍停在 `/faq`；而**直接加载**时 title / description / canonical / og:url 四项全对
- 该现象在同项目 wiki 里有 2026-07-29 的记录，即至少存在了三个月

## 适用范围

- **适用于**：React 18 `hydrateRoot` + 预渲染/SSG + 全站 lazy 路由；Vue 的 `defineAsyncComponent` + Suspense 同理
- **对照产物法适用于任何"这个报错是不是我引入的"问题**，不限前端——只要构建是可重复的
- **不适用于**：框架自带 SSR 且 chunk 与 HTML 同批次下发的方案（Next.js / Nuxt / Remix 的默认链路），它们在服务端就处理了这层

## 相关

- [用源码的写法去校验预渲染产物](prerender-output-verification-false-negatives.md) —— 起静态服务验收时为什么不能用 `serve -s`
- [SPA 模板里硬编码页面级标签](spa-template-hardcoded-page-level-tags.md) —— 同一项目里 canonical 的另一处坑，与本条的"站内跳转后 head 不更新"相邻
- [搭新东西主动暴露老 bug](../patterns/build-new-exposes-old-bugs.md) —— 本条是它的反向情形：新改动暴露老缺陷时，先证明它是老的，再决定管不管
