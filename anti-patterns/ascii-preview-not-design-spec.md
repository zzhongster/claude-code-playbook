# AskUserQuestion 的 ASCII preview 不是设计稿

> 实例：trade-ai #266 草稿箱 tab —— 2026-05-20

## 问题

新增 UI（tab / modal / 子页面）时，用 AskUserQuestion 给用户看 ASCII 草图选 A/B，用户选完后**直接写代码 commit**，跳过设计工具（Pencil / Figma）。

```
┌────────────────────────────────────┐
│ 🔍 搜索框                          │
│  📄 已发布 (148)   ✏️ 草稿 (12)    │ ← tab
│ ────────────                       │
└────────────────────────────────────┘
```

看起来对齐了方向，于是开干。

## 为什么不行

ASCII preview 只能表达"哪两块东西在一起"，**无法表达**：

| 维度 | ASCII | 设计稿 |
|---|---|---|
| 字号 / 字重 | ✗ | fs 11/13 + fw 400/500/600 |
| 颜色 / 透明度 | ✗ | hex / token 精确值 |
| 间距 / padding | 模糊（一格 char ≈ 多少 px？） | px 精确 |
| 复用组件 | 无 | 设计系统 token / reusable |
| 设计师早期预留 | 看不到 | 经常发现"原来这块早就画过"|

## 真实代价

代码写完后查 Pencil，发现**设计师早就有同位置的预留节点**（屏 1 v3 viewTabs 节点 HTFCF），对照规范有 3 处偏差：

- tab padding 上下：写了 8，规范 10
- active 数字：写成 pill 包裹，规范是裸数字 muted
- inactive pill 颜色：写成 subtle bg + text-muted，规范是 primary-tint bg + primary-500 fg fw 500

肉眼接近，**像素错**。后续 PDF 模板复用此 token 时偏差会放大。

## 正解

1. AskUserQuestion preview 只用于"方向 A vs B"高层选择（独立路由 vs query / 严格 vs 宽松定义）
2. 用户答完，**仍走设计工具 first**：
   - 查现有相关屏节点（往往发现预留位置）
   - 复制新增"演进节点"，命名带 issue 号
   - 按节点 spec 字面值写代码，注释 `// Pencil: xxx.pen <node_id>`
3. 触发词清单（任意一条出现就停下查设计稿）：
   - 新建路由 / 新建页面 / 新建 modal
   - 添加 tab / 添加 chip / 添加按钮
   - 子页面 / 内嵌区块 / 上传组件
   - 重排列表 / 改表头列

## 事后追溯（已经写完代码才发现违反流程）

不算"通过"但能保：

1. 承认违反流程（别找借口）
2. 补设计稿（新增演进节点，标注 commit hash 反向追溯）
3. 修代码按 spec 对齐
4. 注释里加节点 ID

## 关联

- 反面：[设计稿优先工作流](../patterns/design-first-workflow.md)（待补）
- 同类：[playwright 黑盒](../patterns/playwright-blackbox-recipes.md) 第 8 节"元素文案陷阱"
