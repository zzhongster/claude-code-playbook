# 反模式：无 BOM 的 UTF-8 .ps1 在中文 Windows 上被 PowerShell 5.1 按 GBK 误读，中文字符串引号被"吃"→连环 ParserError

## 一句话结论

Windows 自带的 PowerShell 5.1 读取**无 BOM** 的 `.ps1` 文件时，按当前系统 ANSI 代码页解码（简体中文系统 = GBK/936），不是 UTF-8。脚本里的中文（UTF-8 是 3 字节/字）被 GBK 按 2 字节错位拆解，会把紧邻的引号 `'`/`"` 吞进乱码里——于是**所有含中文的字符串行引号错位**，解析器连环报"缺少 )"、"缺少 }"、"字符串缺少终止符"。报错行号指向的是中文行，且回显的中文是乱码（如 `未 安装 跳过` → `鏈 香瑁? 璺宠繃`）——这就是铁证。**根因是编码，不是你的括号写错了。**

## 场景

- 给不懂技术的（中文 Windows）同事分发一键安装脚本 `install.ps1` + `install.bat`
- 脚本含大量中文 `Write-Host` 提示，用编辑器/程序以 UTF-8 无 BOM 存盘（Mac/Linux 默认、多数编辑器默认）
- Mac/Linux 或英文 Windows 上测试正常（bash 直读 UTF-8、英文系统 ANSI≈Latin1 不吃 3 字节汉字），一到简体中文 Windows PS5.1 就崩

## 识别（乱码特征 = 秒级定位）

- 报错回显里的中文变成 `鏈`/`香瑁`/`璺宠繃` 这类"生僻字堆"——这是 UTF-8 字节被 GBK 解码的典型形态
- 报错类型集中在 `MissingEndCurlyBrace` / 缺 `)` / `字符串缺少终止符`，且行号全是**含中文的行**
- `.bat` 用 `chcp 65001` 也没用——`chcp` 改的是控制台代码页，`powershell -File` 读 `.ps1` 用的是**文件编码探测**，两回事

## 解法

1. **给 .ps1 存 UTF-8 with BOM**（`EF BB BF` 前缀）。PS5.1 见 BOM 即按 BOM 指示的 UTF-8 解码，根治。Python：`open(path,"w",encoding="utf-8-sig")`；只加前缀、内容字节不变。
2. **`.bat` 保留 BOM + `chcp 65001`**（控制台正确显示中文，与文件编码是两层，都要）。
3. **验证要在目标编码维度做**，不能只在 Mac 上"能跑"就发：
   - 有 PowerShell 时用解析器：`[System.Management.Automation.Language.Parser]::ParseFile($path,[ref]$t,[ref]$e); $e.Count` —— 0 即语法通过
   - 无 PS 环境（如 Mac 无法 sudo 装 pwsh）时的等价静态检查：以 `utf-8-sig` 读入后，逐行验证**引号闭合 + 圆括号平衡 + 花括号平衡 + here-string(@"…"@) 闭合**——精确覆盖 GBK 误读会产生的那三类错误；全闭合即反证"报错是编码引起、非真语法错"
   - 还可正向复现：用 Python `content.encode('utf-8').decode('gbk')` 模拟 PS5.1 的误读，看引号是否错位

## 教训

- 跨平台分发脚本，"在我机器上能跑"是最弱的证据——**目标平台的编码/代码页是独立变量**，必须单独验证。中文 Windows + PS5.1 是最容易被忽略的组合。
- BOM 不是历史包袱：给**中文 Windows 消费的 .ps1/.bat** 就是要带 UTF-8 BOM，这是 PS5.1 的兼容契约。
- 与 [mcp-remote-http-client-gotchas](mcp-remote-http-client-gotchas.md) 同源教训：分发给非技术用户的东西，每个"想当然能用"的环节都要在最苛刻的目标环境实证，否则翻车成本（同事一次次截图、你一轮轮改）远高于验证成本。
