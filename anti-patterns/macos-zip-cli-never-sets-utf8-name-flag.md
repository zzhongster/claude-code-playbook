# 反模式：用 macOS 自带 `zip` 打含中文文件名的包，Windows 解压全是乱码

## 一句话结论

macOS 自带的 `zip`（Apple 改过的 Info-ZIP 3.0）**无论 locale 是什么、带不带 `-X`，都不给条目写 UTF-8 文件名标志位（general purpose flag bit 11）**，也不写 0x7075 Unicode Path 扩展字段。文件名字节其实是 UTF-8，但没有任何标记说明它是 UTF-8。macOS 的 `unzip` / Finder 默认按 UTF-8 猜，本地解压一切正常；Windows 资源管理器按系统代码页（中文系统是 GBK）解，中文名全变乱码。**给 Windows 收件人（财务、客户）发 zip，必须用 Python `zipfile` 打包**，它对非 ASCII 名自动置位。

## 场景

- 用 Claude Code 在 Mac 上生成交付包：报销发票、合同附件、导出报表，文件名含中文
- 收件人在 Windows 上解压（财务、行政、客户）
- 你在 Mac 上 `unzip -l` 或双击解压验过，"看起来没问题"

## 踩坑经历

2026-09-09，月度报销收尾。reimbursement skill 里写的打包命令是：

```bash
( cd "_pack" && zip -r -X "../报销.zip" "报销" -x "*.DS_Store" >/dev/null )
```

上一个会话打出的 `报销.zip` 用 Python 读 namelist 是正常中文，这次按同一条命令重打后，Python 读出来是：

```
µèÑΘöÇ/σÅæτÑ¿/µèÑΘöÇ/09-07 workbuddy∩╝êµèÑΘöÇ┬╖workbuddy ┬Ñ140.00∩╝ë.pdf
```

这是 UTF-8 字节被按 cp437 解码的典型样子（`报` = `E6 8A A5` → `µ è Ñ`）。Python `zipfile` 的规则是：条目带 bit 11 才按 UTF-8 解，否则按 cp437。Windows 资源管理器的规则类似：没 bit 11 就按 ANSI 代码页解。所以 Python 读出乱码 = Windows 上也会乱码。

用一个只含一个中文文件名的目录做了 5 组对照：

| locale | `-X` | bit 11 | extra 字段 |
|---|---|---|---|
| C | 有 | N | 无 |
| C | 无 | N | 0x5455, 0x7875 |
| en_US.UTF-8 | 有 | N | 无 |
| en_US.UTF-8 | 无 | N | 0x5455, 0x7875 |
| zh_CN.UTF-8 | 无 | N | 0x5455, 0x7875 |

`zip -v` 显示 `This is Zip 3.0 (July 5th 2008), by Info-ZIP, with modifications by Apple Inc.`。五组全 N，说明和 locale、`-X` 都无关，是这个二进制本身就不写。`-X` 剥掉的只是时间戳（0x5455）和 uid/gid（0x7875），本来就没有 Unicode Path（0x7075）可剥。

回头看上一个会话那个"正常"的 zip：12 个条目全带 bit 11，且**没有目录条目**（Info-ZIP `zip -r` 默认会加 `报销/`、`报销/发票/` 这样的目录项）。所以它根本不是 `zip` 命令打的，是 Python 打的。skill 文档里写的命令从来没有产出过一个正确的包，只是之前没人用 Python 读过。

## 根因

zip 格式里文件名编码靠两个东西之一声明：general purpose flag 的 bit 11（EFS，"名字是 UTF-8"），或 Info-ZIP 的 0x7075 Unicode Path 扩展字段。两个都没有时，解压器只能猜：

- macOS `unzip` / Finder / `ditto`：猜 UTF-8 → 正常
- Python `zipfile`：按规范退回 cp437 → 乱码
- Windows 资源管理器：按系统 ANSI 代码页 → 中文系统上 GBK 解 UTF-8 字节 → 乱码
- WinRAR / 7-Zip / 360 压缩：有启发式探测，多数时候能猜对，但不保证

Apple 版 zip 3.0 两个都不写。这是个 2008 年的二进制，Apple 的修改没碰这块。

## 修复

用 Python `zipfile`，它对含非 ASCII 字符的名字自动置 bit 11：

```python
import os, zipfile
out_tmp, out = "报销.zip.tmp", "报销.zip"
entries = [("报销明细.xlsx", "报销/报销明细.xlsx")]
for root, dirs, files in os.walk("发票"):
    dirs.sort()
    for f in sorted(files):
        if f == ".DS_Store":
            continue
        src = os.path.join(root, f)
        entries.append((src, "报销/" + src.replace(os.sep, "/")))
with zipfile.ZipFile(out_tmp, "w", zipfile.ZIP_DEFLATED) as z:
    for src, arc in entries:
        z.write(src, arc)
os.replace(out_tmp, out)   # 先写临时文件再原子改名
```

顺手把 `.DS_Store` 排除、不写目录条目、按名排序保证可复现。已固化为 `~/.claude/skills/reimbursement/scripts/pack_zip.py`，打包后自检并打印 `utf8_flag: N/N`。

## 验证方法

**别用 `unzip -l` 或 Finder 验**，它们在 Mac 上永远显示正常。用 Python 读标志位：

```python
import zipfile
z = zipfile.ZipFile("报销.zip")
infos = z.infolist()
print(sum(1 for i in infos if i.flag_bits & 0x800), "/", len(infos))   # 应等于含非 ASCII 名的条目数
print(infos[0].filename)   # 应是可读中文，不是 µèÑΘöÇ
```

只要 Python 打印的名字是可读中文，Windows 上就没问题；打印出 `µèÑΘöÇ` 这种 cp437 风格的乱码，Windows 上一定乱。

## 适用范围

- **适用于**：macOS 自带 `/usr/bin/zip`（Apple 修改版 Info-ZIP 3.0），收件人用 Windows 或任何按规范实现的解压器
- **不影响**：收件人也在 Mac / Linux 上用 `unzip`（会猜 UTF-8）；文件名全是 ASCII
- **同类工具**：`ditto -c -k` 和 Finder 右键"压缩"打出的包同样不带 bit 11（Finder 会写一个 `__MACOSX` 目录，更糟）；Homebrew 装的新版 Info-ZIP 会置位，但别指望收件方环境里有它
- **推广**：任何"给 Windows 用户的、含非 ASCII 文件名的 zip"都走 Python `zipfile` 或明确置位的库，不走 `zip` CLI

## 相关

- [PowerShell 5.1 写 UTF-8 无 BOM 被按 GBK 读](ps51-utf8-no-bom-gbk-mangles-chinese.md)：同一类问题的镜像，一个是写端不打标记，一个是读端不认标记
- 项目：`~/Downloads/网上购票系统-电子发票通知`（发票收纳箱），skill：`~/.claude/skills/reimbursement`
- 故障时间：2026-09-09
