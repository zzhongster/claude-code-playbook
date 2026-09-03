# 反模式：把 MCP 工具的 filePath 参数当成寻址依据——有状态 GUI 后端下它可能只是提示，操作全落在"当前活动文档"上

## 一句话结论

当 MCP 工具背后是一个**有状态的 GUI 应用**（画布编辑器、IDE、设计工具）时，参数里的 `filePath` 未必是寻址依据：工具可能不会为不存在的路径新建文件，而是静默把操作应用到应用的**当前活动文档**上；更麻烦的是，此后你用同一个工具读回内容（`Get`/`read`）看到的全是那份内存状态，**自证自洽、无法证伪**——只有去文件系统看 mtime 和字节数才能发现改动落错了地方，甚至根本没落盘。

## 场景

- MCP 工具带 `filePath` / `path` / `file` 参数，看起来像"对这个文件做操作"
- 但它的后端不是无状态的文件读写，而是一个**打开着某个文档的应用进程**（pen.dev 画布、Figma、Photoshop、IDE 插件、Jupyter 内核……）
- 你连续做十几次调用，每次都返回 `OK`、每次读回的结构都符合预期、截图也正常——直到某个时刻 `ls` 一下，发现目标文件压根不存在

## 踩坑经历

2026-09-03，`trade-agent-data` 项目，用 pen.dev 的 pencil MCP 画一份运营后台设计稿。当时 app 里打开着的是 `docs/design/oauth-pages.pen`（OAuth 授权页设计，3 个 frame，生产在用）。

我全程给 `execute` 传 `filePath: ".../docs/design/admin-console.pen"`，画了 5 个 frame（流程图 + 4 个页面）。**十几次调用全部返回 OK**，`Get` 读回的节点树正确，`TakeScreenshot` 出图正常，创建的节点 id 一一返回。没有任何一处提示"这个文件不存在"。

直到用户问"这个文件我没看到啊？"：

```bash
$ ls -la docs/design/
-rw-r--r--  1 ... 201382 → 252514  Sep 3 16:44  oauth-pages.pen   # ← 体积涨了 5 万字节，mtime 正是我画图的时间
# admin-console.pen 根本不存在
$ git diff --stat docs/design/oauth-pages.pen
 1 file changed, 5278 insertions(+)
```

**5 个 frame 全画进了用户的授权页设计稿里**，连我 `SetVariables` 定义的 10 个设计 token 也一并注入了（该文件原本 `variables` 为空）。

### 补救时踩了第二个、第三个坑

想把两边拆开，于是 `cp oauth-pages.pen admin-console.pen`，打算在副本里删掉 3 个授权页 frame、在原件里删掉我的 5 个 frame。

```bash
$ cp docs/design/oauth-pages.pen docs/design/admin-console.pen
```

对"副本"执行 `Delete` 三个授权页 frame → 返回 OK，`Get` 读回只剩我的 5 个 frame，看起来完美。但：

```bash
$ ls -la docs/design/*.pen
-rw-r--r--  1 ... 252514  17:48  admin-console.pen    # 字节数一模一样
-rw-r--r--  1 ... 252514  17:47  oauth-pages.pen      # 删除根本没落盘
```

**坑二**：非活动文档的修改只停在内存里，不自动保存（应用只自动保存它自己打开的那个文件）。

接着对原件执行删除我那 5 个 frame，结果 `Get` 打印出来的顶层节点是**空的**——3 + 5 = 8 个 frame 全没了。`get_app_state` 确认："The document is empty (no top-level nodes)"。

**坑三**：`cp` 出来的副本，**内部节点 id 与原件完全相同**（`o0VPoD`、`SVXGg`……）。对这类以内部 id 标识文档与节点的应用来说，两个文件不是两份独立文档——我对"副本"发出的删除，作用在了同一份内存文档上。于是两轮删除叠加，把它清空了。

万幸的是坑二救了坑三：因为非活动文档的写入不落盘，磁盘上两个文件都还是完好的 252514 字节。**如果当时用户在 app 里点了一下保存，生产在用的授权页设计稿就没了。**

## 根因（三层，每层单独看都不致命）

| # | 机制 | 后果 |
|---|---|---|
| 1 | `filePath` 不是寻址依据，路径不存在时不报错、不新建，静默 fallback 到当前活动文档 | 改动落在错误的文件上，且全程无警告 |
| 2 | 只有"当前活动文档"会自动落盘，其它文档的修改停在内存 | 你以为改了，磁盘上什么都没发生 |
| 3 | 文档/节点身份由**内部 id** 决定，而不是文件路径 | `cp` 出的副本身份不独立，对副本的操作串台到原件 |

三层叠加的结果是一个**完全自洽的假象**：工具说 OK，读回符合预期，截图能出图。所有验证手段都建立在同一份内存状态上，构不成证伪。

## 怎么早点发现

**第一次写操作之后，立刻去文件系统看一眼。** 一条 `ls -la` 就能在十几次调用之前终止这个错误：

```bash
ls -la docs/design/*.pen     # mtime 和字节数是唯一不受内存状态污染的证据
git status --short           # 新文件该出现在 untracked 里
```

**不要用工具自己的读接口当验证**。`Get` / `read_page` / `get_state` 读的是同一份内存，工具写什么它就读回什么——这跟"用读代替渲染来验证视觉产物"是同一类错误（见 `visual-artifact-verified-by-reading-not-rendering.md`）。

**验证证据必须有区分力**。这次我一度用 `grep -c '01 登录' 两个文件` 来判断删除是否生效——但两个文件内容完全相同，grep 结果一样，这个证据**无法区分"删对了"和"什么都没发生"**。真正有区分力的是字节数和 mtime。（同类问题见 `evidence-without-discriminating-power.md`。）

## 正确做法

1. **新建文档要在应用里新建**，不要指望路径参数隐式创建，也不要 `cp` 一份现成文件当模板——带内部 id 的文档格式，副本的身份不独立。
2. **动别人的文件之前先备份到 scratchpad**。这次 `cp` 出的备份是唯一的兜底：内存文档已经被清空，磁盘文件是靠"没落盘"这个巧合才活下来的，不能指望下次也这么走运。
3. **一次只操作一个文档**，操作前用 `get_app_state` 之类确认当前活动文档是谁；不要同时持有两个内容相同的文档。
4. **破坏性操作（Delete/清空）之前，先用文件系统证据确认前一步真的落盘了**，否则你是在一个不确定的状态上继续叠加操作。

## 适用范围

- **适用于**：所有后端是有状态 GUI 应用的 MCP（设计工具、编辑器、浏览器自动化、模拟器控制、Jupyter/REPL 内核）——凡是"工具进程里存着一份你看不见的当前状态"的场景
- **不适用于**：无状态的文件读写工具（`Read`/`Write`/`Edit`），路径就是寻址，写完即落盘
- **更通用的教训**：**工具返回 OK 只证明请求被接受，不证明副作用发生在你以为的地方**。凡是副作用最终应该体现在文件系统/数据库上的操作，验证就必须去文件系统/数据库看，并且要挑**能区分目标与非目标**的证据——用工具自己的读接口验证工具自己的写操作，永远是自洽的

## 相关

- 项目源：`trade-agent-data`，2026-09-03（`docs/design/oauth-pages.pen` 被误写入 5 个 frame + 10 个变量）
- 同类：[evidence-without-discriminating-power.md](evidence-without-discriminating-power.md)、[visual-artifact-verified-by-reading-not-rendering.md](visual-artifact-verified-by-reading-not-rendering.md)、[tmp-path-hides-rerun-pollution.md](tmp-path-hides-rerun-pollution.md)
