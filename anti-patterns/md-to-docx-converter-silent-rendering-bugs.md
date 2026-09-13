# 反模式：自写 markdown→docx 转换器"能打开"就算过——四类构造静默渲染错

## 一句话结论

用 `docx` npm 包手搓的轻量 md→docx 脚本，遇到作者没预料到的 markdown 构造会**静默出错、文件照常打开**：表头行空白、多个编号列表接着编号、引用块里的 `**加粗**` 原样显示星号、列表项的缩进续行被拆成独立段落且编号全变成"1."。字数对得上、Word 能打开，谁都不会发现——**只有转成 PDF 翻页看才露馅**。两份报告踩了四个，每个都是交付前后才被截图发现。

## 场景

- skill 里自带一个 200 行左右的 `md_to_docx.js`，报告正文是 LLM 生成的 markdown
- 验收判据只有"生成成功 + 文件大小合理"
- LLM 写 markdown 的习惯（长句硬换行、列表续行缩进、引用块里加粗）恰好是简易转换器没覆盖的构造

## 详细说明

| 构造 | 症状 | 根因 | 修法 |
|---|---|---|---|
| 表头单元格 | 表头整行空白 | `new TextRun({ ...r.options, bold: true })`——已构造的 TextRun 实例的 `options` 里没有 `text` | 表头直接用原文重建：`new TextRun({ text: text.replace(/\*\*/g,''), bold: true })` |
| 多个独立有序列表 | 第二个列表从 4、5、6 接着编 | numbering 没传 `instance`，全文共享一个计数器 | 上一个元素不是有序项时 `listInstance += 1`，并传 `instance: listInstance` |
| 引用块 `> …**加粗**…` | 星号原样显示 | 引用块把整行塞进一个 `TextRun`，没走行内解析 | 行内解析函数加 `extra` 样式参数：`inlineRuns(text, { italics: true, color: '555555' })` |
| 列表项的缩进续行 | 编号全是"1."，续行变成独立段落 | 逐行匹配，续行不是列表项；"上一行是否有序项"的判断被续行打断 | 吸收紧随其后的缩进续行并入当前项；用"上一项结束行号"判断连续，而不是看 `lines[i-1]` |

续行吸收的核心逻辑：

```js
function withContinuation(text, j) {
  while (j < lines.length && /^\s{2,}\S/.test(lines[j])
         && !/^\s*([-*+]|\d+\.)\s+/.test(lines[j]) && !/^\s*[|>#]/.test(lines[j])) {
    const nxt = lines[j].trim();
    text += (/[A-Za-z0-9,.;:)]$/.test(text) && /^[A-Za-z0-9(]/.test(nxt)) ? ' ' + nxt : nxt;
    j++;
  }
  return [text, j];
}
```

### 验收清单（每次生成 docx 后）

1. `soffice --headless --convert-to pdf report.docx`
2. `pdftotext -f N -l N report.pdf -` 定位含表格、编号列表、引用块、长列表项的页（写成脚本循环，worktree 会话里内联循环会被沙盒拦）
3. 逐页看图，检查四件事：**表头有字 / 每个列表从 1 开始 / 引用块加粗生效 / 列表续行在同一项里**

## 数据支撑

| 报告 | 发现 | 影响面 |
|---|---|---|
| 印度稀土永磁报告（2026-09-10） | 表头空白、列表连号 | 18 张表全部无表头；第一版已发给用户 |
| 劭鑫智能路亚艇报告（2026-09-13） | 引用块星号、续行拆段 | 65 处列表续行受影响；171 页 |

四个修复目前落在工作副本上；skill 原始资产若不同步，下一份报告会原样再踩一遍。

## 适用范围

- 适用于：任何自写的 markdown → docx / pptx / pdf 转换，尤其是正文由 LLM 生成时
- 不适用于：pandoc 等成熟转换器——但抽页看一眼的成本依然值得

## 相关

- [AI 生成的视觉产物只读代码验收、从不真渲染](visual-artifact-verified-by-reading-not-rendering.md)——本条是它在文档产物上的具体形态
- [无头截图当手机验收结论](render-verification-false-alarms.md)——反方向：渲染工具自己报了假警
- [worktree 会话沙盒按 git 字样拦命令](../patterns/worktree-session-sandbox-git-keyword-workaround.md)
