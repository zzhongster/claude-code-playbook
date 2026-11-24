# Claude 派活给 Codex：用 `codex exec` 当执行位和冷审位

## 一句话结论

Claude 当协调位，用 `codex exec` 非交互地把「写代码」和「只读冷审」派给 Codex。Codex 只改工作区、不提交；它报的「做完了」「测试通过」一律不作数，由 Claude 读 diff、在沙箱外重跑测试、补提交。**退出码 0 只说明它跑到了头，不说明活做成了。**

## 场景

- 一张卡要写的代码量大，想让另一家模型写，Claude 负责拆活、验收、提交、建 PR；
- 提 PR 前要一道**和作者不共享上下文、也不共享模型盲区**的冷审；
- 规格/文档改口径后要多轮复核到零红。

## 配方

以下命令 2026-11-24 在 codex-cli 0.166 上实测过。二进制用 ChatGPT App 自带的那份（`/Applications/ChatGPT.app/Contents/Resources/codex-cli/CodexCLI.app/Contents/MacOS/codex`，下文记作 `$CODEX`）。模型和推理档位读 `~/.codex/config.toml`，要临时换就加 `-m <模型>` 或 `-c model_reasoning_effort="high"`。

| 场景 | 命令 | 要点 |
|---|---|---|
| ① 只读冷审 / 调研 | `$CODEX exec -s read-only -C <wt> --json -o out.md - < prompt.md` | 每轮开**新会话**，别续接写代码的那个会话，否则上下文就不冷了 |
| ② 写代码 | `$CODEX exec -s workspace-write -C <wt> --json -o out.md - < prompt.md` | 在 Claude 建好的独立 worktree 里跑；**它提交不了**（见坑 1），提交由 Claude 做 |
| ③ 续写 / 回修 | `$CODEX exec --json -o out.md resume <thread_id> - < prompt.md` | 沙箱档位**沿用原会话**（只读会话续接后仍写不了），`resume` 不收 `-s`/`-C`；**必须在原目录跑**（默认按 cwd 过滤会话）；exec 级参数如 `--skip-git-repo-check` 要写在 `resume` **前面** |
| ④ 定向复核 | 同 ①，新会话 | prompt 只给「上一轮红项清单 + 修订 diff」，问两件事：每条闭合了没有，有没有引出新矛盾 |

`thread_id` 从 `--json` 事件流的第一行 `{"type":"thread.started","thread_id":"…"}` 里取。续接后 `thread_id` 不变。

### 包装脚本（三条路径实跑过）

```bash
#!/usr/bin/env bash
# codex-run.sh ro|rw <工作目录> <prompt 文件> <产物目录>
# codex-run.sh resume <工作目录> <prompt 文件> <产物目录> <thread_id>
# 产物：out.md · events.jsonl · err.log · thread_id · DONE（退出码 + turn 状态）
set -u
mode=$1 dir=$2 prompt=$3 out=$4 tid=${5:-}
CODEX=${CODEX_BIN:-/Applications/ChatGPT.app/Contents/Resources/codex-cli/CodexCLI.app/Contents/MacOS/codex}
mkdir -p "$out"; rm -f "$out/DONE" "$out/out.md"
cd "$dir" || { echo "exit=2 status=bad_dir" > "$out/DONE"; exit 2; }
common=(--json -o "$out/out.md")
git -C "$dir" rev-parse --git-dir >/dev/null 2>&1 || common=(--skip-git-repo-check "${common[@]}")
case $mode in
  ro)     "$CODEX" exec -s read-only       -C "$dir" "${common[@]}" - ;;
  rw)     "$CODEX" exec -s workspace-write -C "$dir" "${common[@]}" - ;;
  resume) "$CODEX" exec "${common[@]}" resume "$tid" - ;;
  *) echo "exit=2 status=bad_mode" > "$out/DONE"; exit 2 ;;
esac < "$prompt" > "$out/events.jsonl" 2> "$out/err.log"
rc=$?
sed -nE '1s/.*"thread_id":"([^"]+)".*/\1/p' "$out/events.jsonl" > "$out/thread_id"
[ -s "$out/thread_id" ] || [ -z "$tid" ] || echo "$tid" > "$out/thread_id"
st=$(grep -oE '"type":"turn\.(completed|failed)"' "$out/events.jsonl" | tail -1 | sed -E 's/.*turn\.//; s/"//')
echo "exit=$rc status=${st:-none}" > "$out/DONE"
exit $rc
```

实测结果（0.166，1124）：只读、可写、续接三条路径都是 `exit=0 status=completed`；模型不可用是 `exit=1`、`turn.failed`、**不生成 out.md**；二进制路径错是 `exit=127 status=none`；目录不存在是 `exit=2 status=bad_dir`。

### 后台跑，等结束标记

一轮写代码常要 5–30 分钟，Claude 不要前台干等：

1. 用 Bash 的 `run_in_background` 跑 `codex-run.sh`；
2. 用 Monitor 或 until 循环等 `DONE` 出现：`until [ -f out/DONE ]; do sleep 30; done; cat out/DONE`；
3. **别套 `timeout`**：macOS 自带没有这个命令，`timeout 1500 codex …` 直接退 127，那一轮冷审根本没跑（1115 实例）。

## 派活 prompt 怎么写

Codex 拿不到 Claude 的会话上下文，也读不到 Claude 的 hook。prompt 就是它能拿到的全部：

- **为什么做**：一两句，跟判据放在一起（判据随子任务走、理由留在母卡上，是跑偏的主要来源）；
- **范围**：只许动哪些目录；共享文件不许碰，要碰就停下来报；
- **硬规矩**：仓库的约定原文贴进去或点名文件让它先读（`AGENTS.md`、规格目录）；
- **不提交**：写明「只改文件，不要 git commit / push」，省得它花一轮去撞提交权限；
- **交回格式**：改了哪些文件、跑了哪些命令、每条命令的**原始输出**和通过/跳过数、没做完的、偏离了哪里。要它贴原始输出，不要它总结。

冷审 prompt 另加三样：diff 范围（`git diff origin/main...HEAD`）、红线清单、**旧口径词表**（要它对全树 grep，不只读 diff，见坑 6）。

## 收尾核验清单（Codex 每轮交回后，Claude 必做）

1. `git status` + `git diff` 读实物，核范围有没有越界；
2. **在沙箱外重跑测试**，看逐条 passed / skipped 计数，不看 Codex 的报告（见坑 2、3）；
3. 查有没有「为过门改形」（见坑 4）；
4. 提交由 Claude 做，提交信息里写明哪部分是 Codex 写的；
5. 冷审的红项对着卡面裁，不照单全收（见坑 5）。

## 坑

| # | 现象 | 怎么办 | 来源 |
|---|---|---|---|
| 1 | `workspace-write` 在 linked worktree 里 `git commit` 失败：`Unable to create …/.git/worktrees/<名>/index.lock: Operation not permitted`。`--add-dir <主 clone>/.git` 也放不开 | Codex 只改不提交，Claude 提交 | 1124 实测 |
| 2 | 沙箱连不上本机 Docker / Postgres（EPERM），集成测试整片 skipped，它照样报「通过」 | 集成测试一律由 Claude 在本机重跑；第一次实跑红过 24 条，全是夹具问题 | 约 8 份会话归档，1115–1119 |
| 3 | 报的通过数是改动之前跑的（换了模块导出，三个单测里的 mock 没跟着改） | 只认 Claude 自己最后一次跑的数 | 1118 归档 |
| 4 | 为过长度门把整个导出文件压成一行 | diff 里看到格式大面积变化就追问为什么 | 1118 归档 |
| 5 | 冷审定级偏高；把卡面已裁定的取舍也判成 🔴 | 冷审只出发现，裁决在协调位 | 1027 对比、1115 归档 |
| 6 | 只读 diff 查不出「改了口径、别处旧句没跟着改」；开放式冷审两轮各十几条红、不收敛 | 给旧口径词表全树 grep；第三轮起改成定向复核（配方 ④）。一次九轮才零红，先自己 grep 清一遍再派能省掉大半轮次 | 1115、1118 归档 |
| 7 | 任务没做成（提交被拒、写文件被拒）时退出码仍是 0，`turn.completed` 照常出现 | 完成信号只当「跑完了」用，做没做成看实物 | 1124 实测 |
| 8 | `~/.local/bin/codex`（0.144，独立安装的旧版）用 ChatGPT 账号跑报错退 1 | 用 ChatGPT App 自带的那份 | 1027 观察，1124 复核仍退 1 |
| 9 | 仓库的 `.codex/hooks.json` 未提交时，worktree 里的 PreToolUse hook 一道都不响；hook 脚本依赖 `CLAUDE_PROJECT_DIR` | 外部 agent 的产出只能靠 git pre-commit、CI 必需检查和 Claude 冷核兜底 | 1027 观察，**1124 未复核** |
| 10 | Codex 在跑时机器忙，带墙钟预算的测试会超时，main 基线同样红 | 判红前先跑 main 基线对照 | 1118 归档 |
| 11 | stderr 里有大量无关报错（某个 skill 缺 frontmatter、某个 MCP 要 OAuth），不影响结果 | 判成败看 `DONE` 和 events，不看 err.log 有没有 ERROR | 1124 实测 |

## 数据支撑

- **同题冷审对比，只有一张 PR（1027）**：Codex（gpt-6.1-sol，high，6.5 分钟）报 2 条正确性 bug，全真，定级偏高，漏了测试缺口；Cursor（composer-2.5，2.3 分钟）报 4 条，全真，偏测试和文档，漏掉那两条正确性 bug，结论是「批」。两家互补。样本量 1，不足以排名。
- **分工落地（1115–1123，一个 TS monorepo 的十几张卡）**：「Codex 写、Claude 收尾」，加「Codex 只读冷审 + Claude 冷读 + 按路径的专项 auditor」。反例库里有十余条是 Codex 冷审或自查发现的（死锁、幂等基线、回填空值语义、租户设置校验等）。
- **1119 起该项目停用 Cursor**：写代码交给 Codex 或 Claude 子 agent（各用独立 worktree），冷审用 Codex 只读新会话。Cursor 带 `--force --trust` 跑长任务曾连断两次（HTTP/2 keepalive），续写要先让它按 `git status` 盘点再补。

## 适用范围

- 适用于：Claude Code 当协调位，本机装了 ChatGPT App 带的 codex CLI、用 ChatGPT 账号登录；有 CI 必需检查兜底的仓库。
- 不适用于：需要连本机数据库、Docker、内网服务才能验证的活（沙箱连不上，只能交回 Claude 跑）；要 Codex 自己提交、推送、建 PR 的流程（worktree 里提交不了，也不该让它碰）。
- 验证面：上述实据都来自**同一个仓库**，加 1124 在临时仓库里的命令实测。CLI 参数随版本变化快，换版本后先跑一遍包装脚本的三条路径。

## 相关

- [fresh-context-cold-review-before-pr](fresh-context-cold-review-before-pr.md)：冷审为什么要干净上下文
- [whole-branch-review-catches-emergent-defects](whole-branch-review-catches-emergent-defects.md)
- [cross-project-task-handoff](cross-project-task-handoff.md)
