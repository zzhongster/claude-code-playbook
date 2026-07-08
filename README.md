# Claude Code Playbook

用 Claude Code 编程的工程实践手册。不是功能列表，不是 tips 汇总，而是**经过实战验证、带数据、可复用的工程决策**。

## 为什么做这个

市面上的 Claude Code 资源多是功能罗列型（命令大全、awesome 清单）。但真正影响效率的不是知道多少命令，而是**工程决策**——什么时候用子 Agent、什么时候单对话、怎么写 prompt 让 LLM 不偷懒。

这些决策需要实战验证，需要数据支撑，需要踩过坑才能总结出来。

## 内容结构

```
claude-code-playbook/
├── foundations/       ← 基础共识（官方 + 社区公认的实践）
├── patterns/          ← 工程模式（经过验证的做法）
├── anti-patterns/     ← 反模式（踩过的坑）
├── experiments/       ← 对比实验（带数据的验证）
└── templates/         ← 可复用模板
```

### Foundations — 基础共识

| 文档 | 主题 |
|------|------|
| [CLAUDE.md 写法](foundations/01-claude-md.md) | 项目入口文件的最佳实践 |
| [先规划再写码](foundations/02-plan-before-code.md) | Plan Mode 的正确用法 |
| [给验证手段](foundations/03-verification.md) | 测试、截图、lint — Claude 能自我验证才能自主工作 |
| [上下文管理](foundations/04-context-management.md) | 窗口有限，如何高效利用 |
| [Hooks 做确定性保障](foundations/05-hooks-and-guards.md) | CLAUDE.md 是建议，Hooks 是强制 |
| [并行工作流](foundations/06-parallel-workflows.md) | 子 Agent、worktree、batch 的选择 |

### Patterns — 工程模式

| 文档 | 核心结论 |
|------|---------|
| [Prompt > 模型 > 架构](patterns/prompt-over-model.md) | 同一模型优化 prompt 的效果 > 换更贵的模型 |
| [单对话 vs 多 Worker](patterns/single-conv-vs-multi-worker.md) | 需要全局意识的任务用单对话，独立任务才拆 |
| [四阶段搜索工作流](patterns/four-stage-search.md) | 广搜→追线索→覆盖检查→输出 |
| [SSH config 当全局机器别名层](patterns/ssh-config-as-global-host-alias.md) | 把 user/IP/key 沉淀到 `~/.ssh/config`，项目和 AI 会话只引用别名 |
| [飞书 vs 企微 机器人推文件选型](patterns/feishu-vs-wecom-bot-file-push.md) | 只推用企微群机器人最省事；私聊/双向/动态IP 用飞书更灵活 |
| [浏览器内无副作用验证](patterns/in-browser-side-effect-free-verification.md) | monkeypatch `fetch`/`Blob` 验离线分支和下载内容，不真断网、不落盘 |
| [openpyxl 填模板保图保格式](patterns/openpyxl-fill-xlsx-template-preserve-images.md) | load→只改单元格→save，无损保留二维码/合并格/字体，别重建 xlsx |
| [微信公众号文章抓取](patterns/wechat-mp-article-scrape-via-micromessenger-ua.md) | curl 带微信 iPhone UA 绕过"环境异常"反爬，bs4 取 js_content |
| [CLAUDE.md 文件夹收件箱](patterns/claude-md-folder-inbox.md) | 给杂物文件夹配 CLAUDE.md 固化分类/命名/解析规则，丢文件即自动归档 |
| [整分支终审抓涌现缺陷](patterns/whole-branch-review-catches-emergent-defects.md) | per-task 评审看不到跨任务/规模涌现缺陷，合并前必做「整分支+最强模型」终审 |
| [反爬 PDF 本地 OCR 修复](patterns/anti-scrape-pdf-local-ocr-repair.md) | 显示正常复制乱码=字体混淆；PDFKit 3x 渲染+Vision OCR 零成本认回，"的"字频<1.5% 判乱码 |
| [证据-结论分离抽取架构](patterns/evidence-conclusion-separation-for-llm-extraction.md) | LLM 只抽证据（引句+出处），矛盾走裁决层，合成纯脚本重放——可溯源、可增量、便宜模型可用 |

### Anti-patterns — 反模式

| 文档 | 踩坑场景 |
|------|---------|
| [碎片化多 Worker](anti-patterns/fragmented-workers.md) | 拆太碎导致每个 Worker 都不够聪明 |
| [便宜模型做判断](anti-patterns/cheap-model-judgment.md) | 省钱模型在筛选/判断任务上翻车 |
| [带本地 daemon 的 Skill 不配套关闭](anti-patterns/skill-with-local-daemon.md) | 装了带后台进程的 skill 不写 stop 脚本=本地后门 |
| [飞书机器人推文件权限踩坑](anti-patterns/feishu-bot-file-upload-scope-gotchas.md) | im:resource:upload 要加在"应用身份"且必须发版本审核才生效 |
| [表头模糊匹配子串方向陷阱](anti-patterns/fuzzy-header-match-substring-direction.md) | 别名⊆表头会误吞长表头；正解两遍匹配+精确占列排除 |
| [微信图文数据藏在图片里](anti-patterns/wechat-mp-data-in-images-not-text.md) | 计划表/分数线常是图片，文字可能与图片矛盾，以图片为准 |
| [同构文档合并整体读=模板化幻觉](anti-patterns/merged-homogeneous-docs-template-hallucination.md) | N 篇相似文档拼一起读，后半会被脑补成统一模板，应逐文件 grep |
| [批量整理票据默认全是正常票](anti-patterns/invoice-batch-assumes-all-normal.md) | 退票费/红冲/折扣/跨年票混在批次里，统一正则会崩或把退票费当票价汇进总额 |
| [DNS fallback 走代理致健康检查死锁](anti-patterns/mihomo-dns-fallback-deadlock-healthcheck.md) | Clash/mihomo fake-ip 下国外 DoH fallback 走代理→url-test 死锁→节点全标 Error→"用不起" |
| [REALITY 伪装目标升级 TLS 打挂全员握手](anti-patterns/reality-fronting-domain-tls-upgrade-breaks-handshake.md) | www.microsoft.com 升 PQ TLS 后不能再当 REALITY 偷证书目标→全员直连挂→像 IP 被封但换目标(apple)即恢复；localhost 自测定位 |
| [吞 stderr 把缺工具伪装成空数据](anti-patterns/silencing-stderr-hides-missing-tool-as-empty-data.md) | 诊断命令加 2>/dev/null 把 command-not-found 吞掉，空输出被当真实零值，根因判断走偏 |
| [GUI 客户端接远程 http MCP 两坑](anti-patterns/mcp-remote-http-client-gotchas.md) | Claude Desktop/WorkBuddy 用 mcp-remote：npx 必须绝对路径(app PATH 精简致 ENOENT)+ 加 --allow-http(拒非 HTTPS)；把命令拎到终端跑定位 |
| [自动化浏览器假象当应用 bug](anti-patterns/browser-automation-env-false-negatives.md) | 后台 rAF 停转/CDP 无原生双击/缓存旧页/headless WebGL 空白——先 console 探针证明事件到达，再谈改代码 |

### Experiments — 对比实验

| 文档 | 实验内容 |
|------|---------|
| [Skill vs Pipeline 全量对比](experiments/2026-03-12-skill-vs-pipeline.md) | 4 种架构方案的完整数据 |

## 内容来源

- **一手实验**：自己的 Claude Code 项目实战（带数据）
- **官方文档**：[Anthropic Best Practices](https://code.claude.com/docs/en/best-practices)、[How Anthropic Teams Use Claude Code](https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf)
- **社区精华**：从高星 repo 和文章中提炼共识

## 使用方式

1. **新项目启动前**：读 foundations/ 建立基础
2. **遇到架构决策时**：查 patterns/ 找参考
3. **踩坑后**：查 anti-patterns/ 看别人是不是也踩过
4. **想验证某个假设时**：参考 experiments/ 的实验方法

## 贡献

欢迎提交你的实战经验！详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## License

MIT
