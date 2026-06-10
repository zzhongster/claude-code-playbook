# openpyxl 复制模板填表，无损保留二维码/图片/格式

## 一句话结论

要把数据填进一个带二维码图片和复杂格式的 Excel 模板，别自己拼 xlsx——用 openpyxl `load_workbook → 只改目标单元格 → save`，3.1.5 实测能无损保留内嵌图片（PNG）、合并单元格、字体/颜色等全部格式。

## 场景

- 客户给一个固定的 Excel 表单模板（进仓单 / 报关单 / 发票），要求程序按行数据批量填，且保留模板里的二维码、logo、说明文字、边框样式。
- 常见误区：以为 openpyxl 打开再保存会丢图片，于是绕去用 xlsxwriter 重建、COM 自动化、或换语言库——其实没必要。

## 详细说明

做法：

```python
import openpyxl
wb = openpyxl.load_workbook("模板.xlsx")   # 直接 load，不要 data_only（要写值）
ws = wb.active
ws["B3"] = "SMAHBDK001-3027464"           # 只改要变的单元格
ws["D4"] = 3027464                         # 货号写成数字就用 int
ws["B7"] = some_datetime
ws["B7"].number_format = 'm"月"d"日"'      # 日期要显示成中文需显式设格式
wb.save("输出.xlsx")
```

关键点：

- openpyxl 3.1.5 加载时会把图片解析进 `ws._images`，save 时写回——图片、drawing、锚点都在。
- 日期单元格：模板里若存的是序列号 + `General` 格式会显示成数字；写入 `datetime` 后要**显式设 `number_format`**。
- 只改目标格、别动其它——固定内容（电话、说明、二维码）留在模板里，每次不碰。
- 定位写入格别写死地址：用"找标签 → 写相邻格"更抗模板变化（见相关 anti-pattern）。
- 先准备"空白模板"：从一份填好的样例里清空变量单元格另存一次，得到可复用的干净模板。

## 数据支撑

进仓单模板（含 `xl/media/image1.png` 二维码 + `image2.png` logo + `drawing1.xml`）round-trip 实测：

| 项 | load → 改 B5 → save 后 |
|---|---|
| 内嵌图片 | 2 张全保留（含锚点位置）|
| 合并单元格 | 8 个全保留 |
| 字体格式 | 红色粗体 20 号标题保留 |
| 单元格值 | 修改生效 |
| 体积 | 74756 → 61834 bytes（重压缩，内容无损）|

校验技巧：用 `zipfile` 打开输出 .xlsx，断言 `xl/media/` 下图片文件仍在，作为"没丢图"的自动化测试。

## 适用范围

- 适用于：填充固定 Excel 表单模板、保留图片/格式、批量生成。
- 不适用于：要新增/移动图片位置（openpyxl 改图能力弱）；超大文件性能敏感场景；`.xls` 老格式（用别的库）。

## 相关

- `anti-patterns/fuzzy-header-match-substring-direction.md`（配套：定位列/标签用名称匹配而非写死坐标）
- 首次验证项目：凯越进仓单自动生成工具（kaiyue-warehouse-tool）
