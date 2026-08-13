# 审批通知与催审升级链：GitHub Actions → 企微群 @到人

> 来源：TOUCH 2.0 审批链自动化（2026-08-13/14，TOU-347）

## 分工判断（先想清楚再选载体）

- **节拍**（每天 21:30、每周一 8:00）→ Claude Code 云端 routine（要 LLM 组织内容）；
- **事件**（PR 提出、review 提交）→ GitHub Actions：秒级触发、零 token 成本、模板化消息不需要 LLM；
- routine 最小间隔 1 小时且每次跑完整会话，拿来做事件响应是杀鸡用牛刀。

## 三类消息，一个 workflow 文件

1. **待审批**：受管路径 PR `opened/ready_for_review` → @首审人 + 链接。**只 @ 流程定义的首审人**，不 @ CODEOWNERS 点名的所有人——四个人同时被强提醒等于没人负责；
2. **审批结果**：`pull_request_review submitted` → approved【已批准，可以合了】/ changes_requested【要改 + 审查正文首句】@作者。「一行能懂」不需要 LLM：审查文化好的仓库首句就是结论，摘首句截 80 字符即可（按码点截，防 UTF-8 半字）；`commented` 不推（噪音）；
3. **催审升级链**：`schedule` 每 15 分钟扫「被点名审查且零 review」的 open PR——30 分钟无响应转二线、3 小时转兜底人。

## catch 过的设计要点

- **@到人必须用企微 userid**：`mentioned_list` 塞全名不解析、只显示纯文本，实测两次对照才定案。上线前先用真 userid 发一条测试验证强提醒；
- **幂等用 label**：每档催审发送后给 PR 打 label（如 nudge-30m/nudge-3h），扫描时跳过已打标的——每档只响一次不刷屏。label 要**预建**（`gh pr edit --add-label` 不会自动建）；3 小时档补打两档 label，封住「夜间攒到直跳 3h、次日又补 30m」的口子；
- **夜间静默双闸**：cron 只排工作时段（UTC 换算），job 内再按本地时区小时数拦一道——晚间提的 PR 允许次日审，深夜催审违背时限设计本意；
- **「无响应」的代理判据**：`reviewDecision == REVIEW_REQUIRED` 且 requested reviewers 非空。零批准路径的 PR 没有点名，天然不进扫描，不用自己做路径匹配；
- **paths 过滤以 CODEOWNERS 为准，别抄二手表**：冷审时比对发现文档里的「合并规则表」落后权威源两条路径——照表抄的 workflow 恰好会漏掉「被要求批准却收不到通知」的 PR，正是它要消灭的场景；
- **安全面**：permissions 只在需要写 label 的 job 提级（pull-requests/issues: write）；全程不 checkout、不用第三方 action；PR 标题等不可信数据只经 env → shell 变量 → `printf` 参数位，绝不经 `${{ }}` 内插进脚本（防注入）；webhook 走 repo Secret。

## YAML 手感

`run: |` 块标量里写多行 shell 字符串时，续行顶到第 1 列会把块截断（缩进小于块基准即块结束）——多行消息用 `printf '%s\n%s'` 组装。提交前 `python -c "yaml.safe_load(...)"` + `bash -n` 各验一遍，本地十秒钟，省一次红 CI。
