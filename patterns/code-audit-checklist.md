# 大块工作完成后的代码审计标准

> 每次完成一个里程碑 / PR / Epic 后，**主动**用 code-reviewer agent 审计。不要等用户提醒。
>
> 这份 checklist 是 code-reviewer agent 的输入清单，也是人审 PR 的对照表。

---

## 触发时机

| 场景 | 是否触发 | 备注 |
|---|---|---|
| 完成一个 milestone（M1/M2/...） | ✅ 必须 | 即使已 merge，也要补做 |
| Merge 一个 sub-issue PR | ✅ 必须 | 在 merge 前更好 |
| 完成一个 feature 分支 | ✅ 必须 | push 前 |
| 单 commit 小改动 | ⛔ 不需要 | 浪费时间 |
| 仅文档/测试 | ⛔ 不需要 | 除非测试本身复杂 |
| 重构性大改（>200 行 diff） | ✅ 必须 | 即使无新功能 |

**原则**：>50 行业务代码、>1 个文件、跨模块改动 — 都要审。

---

## 8 大审计维度

### 1. 与 Plan / Issue 验收标准的吻合度

- [ ] 验收 checklist 全勾掉？哪些没勾原因？
- [ ] 范围是否扩大（scope creep）？是否有未记录的"顺手改"？
- [ ] 是否引入新的 TODO / FIXME 没归档到 issue？

### 2. 业务逻辑零回归（重构类必查）

- [ ] 旧 API 端点行为是否完全保留（输入输出 / 状态码）？
- [ ] 旧 URL 是否能正确重定向（301 不是 302，避免 SEO 问题）？
- [ ] 数据库字段读写路径是否影响（schema 变更需 migration）？
- [ ] 关键交互（保存 / 编辑 / 删除）能否端到端跑通？

### 3. 死代码 / 冗余

- [ ] 重构留下的旧函数是否还被引用？grep 一遍验证
- [ ] 注释掉的代码是否清理（用 git history 找回，不留尸体）？
- [ ] partial / 组件抽出后，原文件是否还有重复逻辑？
- [ ] 新增的工具函数是否真的被调用？没人用就删

### 4. 安全性（OWASP Top 10）

- [ ] 用户输入是否 escape（XSS / SQL injection）？
- [ ] 文件上传 / 下载是否限制 path traversal？
- [ ] 新增 API 是否有权限校验（require_permission / get_current_user）？
- [ ] 敏感数据（token / 密码 / 个人信息）是否进了日志？

### 5. 性能 / 资源

- [ ] N+1 查询？eager load 用对了吗？
- [ ] 长循环里有 await 调外部 API？是否要并发或限速？
- [ ] 大文件 / 长文本是否切片处理？
- [ ] 缓存策略是否合理（CSS/JS 静态资源加 hash 或 ETag）？

### 6. 可读性 / 风格

- [ ] 函数名 / 变量名 / 类名是否表意（没 `do_thing` / `tmp` / `data`）？
- [ ] 文件长度是否合理（>500 行考虑拆分）？
- [ ] 注释是否解释 "WHY" 不是 "WHAT"（项目 CLAUDE.md 要求）？
- [ ] 缩进 / 命名 / import 顺序是否符合项目惯例？

### 7. 测试 / 可验证性

- [ ] 关键路径是否有自动化测试（unit / integration / e2e）？
- [ ] UI 改动是否有 Playwright smoke test？
- [ ] 新增 API 端点是否在测试覆盖内？
- [ ] 测试是否真的会 fail（不是 always pass）？

### 8. 文档 / 状态同步

- [ ] CLAUDE.md / README / API doc 是否需要更新？
- [ ] GitHub Issue 验收 checklist 是否同步勾掉？
- [ ] 跟踪 issue（如 #176 设计稿挂接图）是否更新？
- [ ] 提交 message 是否符合规范（feat/fix/refactor + #issue + 简洁描述）？

---

## 调用 code-reviewer agent 的 prompt 模板

```
任务：审计 [PR/分支/commit 范围]

**这次做了什么**（用 1-3 句话告诉 agent）：
- M1 #178 拆 investigation.html 单页 → 主框架 + 7 partial + 屏 4 topBar
- 关键 commits：<sha1> <sha2>
- 验收文档：[GitHub issue 链接 / plan 文件路径]

**对照这份审计标准**：
~/Developer/claude-code-playbook/patterns/code-audit-checklist.md

**重点关注**（≤3 项）：
1. 业务逻辑零回归（旧 URL 重定向、API 兼容）
2. 死代码（旧 inline style 是否还有残留）
3. UI 烟雾测试是否覆盖关键路径

**输出格式**：
- 按 8 维度打分（✅ 通过 / ⚠️ 注意 / ❌ 阻塞）
- 阻塞项必须说明影响
- 给出 5 条以内可执行修复建议（带文件:行号）
- ≤500 字
```

---

## 输出处理

agent 返回审计报告后：

1. **❌ 阻塞项** → 当场修，不能放任合并
2. **⚠️ 注意项** → 评估后，放下一个 PR 或开 follow-up issue
3. **✅ 通过项** → 不动

修完后，重跑一次 audit 验证（用之前的 prompt 加一句「上次审计提到 X，现已修复，请确认」）。

---

## 反模式

### ❌ 等用户提醒才审

> 用户：「改完了吗？」 我：「改完了！」 用户：「你审了吗？」

教训：**主动触发**。每个 PR / 每个 milestone / 每个 feature 完成 = 立即开 audit。

### ❌ 自己审自己

LLM 自己看自己写的代码，盲点跟原作者一样。**必须用独立 agent**（code-reviewer），输入只看 diff + 规范，不带写代码的上下文。

### ❌ Audit 报告全 ✅ 也照样合并

如果 agent 报告 100% 通过，先怀疑 prompt 不够具体。补充「具体 grep 检查 X / Y / Z」再让它跑一遍。

### ❌ Audit 通过但没合并 issue 关联

记得 PR commit message 带 `Closes #N`，否则 issue 不会自动闭环。

---

## 关联

- 项目要求：trade-ai `CLAUDE.md` 「Phase 3: Dev ↔ QA 循环」
- 工具：`superpowers:code-reviewer` agent（Claude Code subagent）
- 触发器：每次 PR / milestone / merge 完成
