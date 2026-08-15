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
| [配置层确认 ≠ 端到端验证](patterns/split-config-check-from-e2e-verification.md) | 回读配置值只证明"写对了"不证明"生效了"；验收拆两条，端到端那条若依赖别人就别挂自己卡上 |

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
| [实时回调同步干重活，重试状态跨 Attempt 复用](anti-patterns/realtime-callback-blocking-and-global-attempt-state.md) | 回调只复制 primitive 快照并有界投递；readiness、写入确认和取消必须按 attempt 隔离 |
| [未抽样验证就照单全收 lint 报告](anti-patterns/trust-linter-output-without-sampling.md) | wiki lint 报 147 条"悬空链接"实为 0 条真悬空（工具不认 \|别名 语法）；批量修复前先抽 3-5 条验证，自引用也报错=最强信号 |
| [SEO 基建只验本地产物不验线上](anti-patterns/seo-verified-in-build-never-in-production.md) | 预渲染在 build/ 里验证通过却从未部署，静默失效 4 个月无人察觉（线上对爬虫仅 649 字节）；内容页排名 1.3–2.0、5990 展示只换 16 点击。验收只认 curl 线上 URL |
| [SPA 模板里硬编码页面级标签](anti-patterns/spa-template-hardcoded-page-level-tags.md) | index.html 是全路由共用模板，写死 canonical=每个子页宣称"首页才是权威版本"、放弃独立收录；不影响任何可见功能故潜伏极久。判据：每页恰好 1 条且指向自己 |
| [忽略预渲染序列化会重排属性](anti-patterns/prerender-serialization-reorders-attributes.md) | puppeteer 输出的是序列化 DOM，meta 属性按字母序重排；站长平台校验是固定顺序正则→验证失败且不给原因。逐字符比对，别只看"标签在不在" |
| [DNS 写入成功≠解析生效](anti-patterns/cloudflare-silent-dns-record-rejection.md) | Cloudflare 静默丢弃重复标签名（send.send.x.com）：API 返 success、UI 显示、GET 读得回，区文件不写→NXDOMAIN。探针二分 30 秒定位根因 |
| [视觉产物只读代码验收不真渲染](anti-patterns/visual-artifact-verified-by-reading-not-rendering.md) | CC 生成的设计稿注释里 `--font-*/--space-*` 的 `*/` 提前闭合注释，吞掉整个 `:root`、119 个 token 全失效；CSS 加载成功、排版还在，编码/括号/变量比对全绿——连去注释的正则都犯同一个错。判据必须写"附渲染截图" |
| [tmp_path 测不到重跑污染](anti-patterns/tmp-path-hides-rerun-pollution.md) | 每个用例发一个空目录=模拟一个永远不会有第二次的世界。225 测试全绿，真实重跑第二次切出新旧混杂 7 个文件、序号还撞车。产出路径由输入决定的功能，要专门造"地面是脏的"用例 |
| [未定义的 CSS 变量让整条声明作废](anti-patterns/undefined-css-var-kills-declaration.md) | 删了 `--accent-fg`，另一个页面还在引用→进度条填充变透明，进度在跑但看不见。不报错不回退，整条声明被丢弃；活过两轮审查，因为审查视野边界是 diff，被删对象的消费者不在 diff 里 |
| [指标绿的原因不是你以为的](anti-patterns/green-metric-measures-wrong-mechanism.md) | 用 `scrollWidth > clientWidth` 查布局破版返回 false——因为文字被挤成竖排、纵向坍缩把横向溢出吸收了。选指标要问"故障发生时它会变成什么"，答案是"正常范围"就换指标 |
| [正则给结构化数据脱敏遮错对象](anti-patterns/regex-redaction-on-structured-data.md) | 在 `{"key":"PASSWORD","value":"<真密码>"}` 上正则命中的是**标签**不是值——输出满屏 `<已遮蔽>` 而密码是明文，且同一份输出里既有遮对的也有遮漏的，肉眼分不出。脱敏没有反馈信号，所以"能不取的就不取"优于"取了再遮" |
| [定时任务只配失败告警](anti-patterns/scheduled-job-only-alerts-on-failure.md) | 三类失败：跑了失败／**根本没跑**／跑了成功但产物是空的。失败告警只覆盖第一类，而后两类是静默的。看门狗必须独立于被监控任务，判断依据要看**产物**不是运行记录；且每类告警都得人为触发过一次 |
| [报错建议的修法把守卫关掉了](anti-patterns/error-message-suggests-the-fix-that-silences-it.md) | `pg_dump` 因 RLS 拒绝导出（退出码 1，好的失败），报错说"会被 RLS 影响"→顺手加 `--enable-row-security`→**退出码 0、dump 里 0 行数据**，结构完整数据全无，下游"恢复演练"还因为数表不数行而放行。看到报错里的选项名先问：它让我读得全，还是让我不再被告知读不全 |
| [「X 点前触发义务」的规则会在 X 点前制造真空](anti-patterns/deadline-triggered-obligation-vacuum.md) | 「19:00 前被要求修改 → 当天改完」惩罚的是**早提 PR**——理性选择变成憋到明早再提，分支多活一夜、审查队列堆到早上，**规则达成了它想防止的事**。挪时点无效（改 18:00 只把规避提前到 17:00），解法是拆开**响应**（30 秒的事，可硬性要求）与**完成**（取决于改动大小，改为自己承诺+承诺有约束力）。AI 优化「逻辑自洽」，人优化「明天同事会怎么做」——规则的判据是激励相容，不是自洽 |
| [串行资源上的并发申领](anti-patterns/concurrent-claims-on-serial-resource.md) | 两个 PR 各自从迁移 `0003` 开始编号，双方本地全绿、两个 CI 也全绿——CI 只跑「本分支 vs main」，从不跑「两个待合分支互相」。冲突在**两份 diff 的交集**里，不属于任何一个 PR，**单 PR 视角结构性不可见**。drizzle 的 snapshot 是链式的，后合方无法手工解、必须整个重生成。同类：ADR 编号、changelog、`_journal.json`、错误码。只要 open PR > 1 且碰同一目录就横向比一次 |
| [照文档配事件钩子做埋点](anti-patterns/hook-instrumentation-double-fires-silently.md) | 三处会错全都不报错：文档说 matcher 是 `Task` 实为 `Agent`（永不触发，3 个对象零记录）；人手敲命令时两个事件**各触发一次**（人机比例直接翻倍，**数字看着完全正常**）；有的客户端把 `/命令` 就地展开、不产生工具调用（只在人这一侧漏，结论恰好符合直觉因而没人质疑）。漏记还有个「零」让人起疑，翻倍什么都不留——配置前先装 dump 探针看真实事件 JSON，入参结构从 `~/.claude/projects/*.jsonl` 历史调用里捞频次（50 带 / 8 不带，只看样例会以为是必填） |
| [改另一个副本的 hook 脚本，当前会话永远加载不到](anti-patterns/hook-edits-in-parallel-clone-never-load.md) | hook 路径锁在会话启动时的 `CLAUDE_PROJECT_DIR`，`cd` 到另一个 clone/worktree 不改变它。失败现象与「我代码写错了」**四项全同**（退出码/stdout/stderr/落盘结果），而你刚改完代码、第一嫌疑人必然是自己——离线用例全绿本该排除它，但直觉会解读成「用例没覆盖真实情况」，**于是回去改一份正确的实现**。排查顺序反过来：先在脚本顶部插一行 marker 确认加载的是哪个文件，再怀疑代码。多副本并行下高频 |
| [报「卡在别人那儿」之前不回查](anti-patterns/blocked-on-what-already-happened.md) | 一份「五件事卡在你这儿」的清单，逐条核实**只有一件成立**：两件已批准、一件路径根本不需要批准、一件前一天已裁。**沉默有三义**——已处理／不需处理／真没人处理——在报告者一侧完全同形，而处置相反（去合／自己动手／才是去催）。真正的阻塞常在自己手上（本例是他自己两条 PR 的 CI 红着），因为它不以「有人欠我」的形式出现所以不进清单。反面同源：给这条坑写条目时用工单号命名分支、正文没写 `Part of`，合并即把一张一件没做的卡标成 Done（时间戳差 2 秒）——**状态信号靠命名巧合传播，必然有时对有时错且错时无声** |
| [把「结果断言」读成「实现机制」](anti-patterns/assertion-read-as-mechanism.md) | 规格里两类句子：**给形状的**（默认值、ID 前缀、字段何时铸造）和**给结果的**（「重复调用不产生新行」）。后者常在多种机制下都成立——本例那条在两种模型下都真，其中一种还更强，**它本来就没规定机制**，所谓「三本同档分册打架、裁决顺序帮不上」是定性错了。同一次里第二处同根：「要动别人刚合入 main 的代码，风险大」也是从措辞推的，一条 `git grep` 证伪——那个函数**零调用方**。三问判别法：形状还是结果／断言在对方机制下成不成立／每句「要动 X」核过没有。裁决还要点名**哪些旧注释的理由在新模型下方向反了**（本例 savepoint 那段：原本必须删的那行，现在删了就是把用户的草稿删掉） |

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
