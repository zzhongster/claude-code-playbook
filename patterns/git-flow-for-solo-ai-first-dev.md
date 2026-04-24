# Git Flow for Solo + AI-First Development

## 背景

非一线开发（产品 / PM / 有业务判断力但不手写 git 命令的人）用 Claude Code 做全栈开发时，需要一套**规范的分支策略 + 版本号管理**，解决两个核心痛点：

1. **hotfix 和新功能开发互不干扰**：生产出 bug 需要紧急修，但本地积攒了一堆未测试的 feature commits。如果直接推会带上未审计代码。
2. **可追溯性**：生产在跑哪个版本？某个 bug 从哪个 commit 引入？业务员反馈 bug 时如何对应到具体改动？

## 适用场景

- 单人 / 小团队 + AI 协作
- 有一个生产环境（单机 or 小集群）
- 发版频率从每日到每周
- 业务员直接使用，不能频繁宕机

## 核心设计

### 分支结构（Git Flow 简化版）

```
main              ← 生产（永远对应线上在跑的）
  ├── hotfix/*    ← 从 main 拉，紧急修，merge 回 main + cherry-pick 到 develop
  └── tag vX.Y.Z  ← 每次部署打 tag

develop           ← 下个版本的集成分支（AI 产出的 feature 都先进这里）
  ├── feat/*      ← 每个功能一个分支，merge 到 develop
  └── release/x.y ← 准备发版时从 develop 拉，只修 bug
```

**核心规则**：
- `main` 受保护，只接受 `release/*` 和 `hotfix/*` 的 merge
- `feat/*` 永远进 `develop`，不直接上生产
- 所有部署都通过 tag（`git checkout v0.2.0`），不通过分支名

### 版本号（SemVer）

`vMAJOR.MINOR.PATCH`：
- MAJOR：DB 破坏性改 / API 协议变 / 重构架构
- MINOR：新功能，向后兼容
- PATCH：bug 修复、安全补丁

### 应用内版本自查

在代码里嵌入版本信息，让运行时可查：

```python
# app/__version__.py
__version__ = "0.1.0"

def _read_commit():
    # 优先从 env（部署注入），其次从 git subprocess
    env = os.environ.get("APP_COMMIT") or os.environ.get("GIT_COMMIT")
    if env: return env[:7]
    try:
        out = subprocess.check_output(["git", "rev-parse", "--short", "HEAD"], ...)
        return out.decode().strip()
    except Exception:
        return "unknown"

__commit__ = _read_commit()
```

**暴露路径**：
- `GET /api/version` → `{version, commit, started_at}` — 运维/监控/UI 都用
- 页面右下角页脚：`v0.1.0 · abc1234`
- 错误告警含版本信息

**Docker 场景**：容器内通常没 `.git`，需要 build 时 `ARG APP_COMMIT` 注入或 compose `environment: APP_COMMIT: ${APP_COMMIT}`

## 实施方式（非技术负责人的使用界面）

关键洞察：**负责人不敲 git 命令，只"口头下达"，AI 执行 + 把关**。

### 五个日常场景

| 场景 | 说什么 | AI 做什么 |
|---|---|---|
| **新功能** | "加 XX 功能" | 从 develop 开 `feat/xxx` → 开发 → PR 回 develop（不上生产）|
| **发版** | "发版" 或 "上线 develop" | 建 `release/0.x` → 让你 review → bump 版本号 + CHANGELOG → tag + 部署 |
| **线上 bug** | "生产 XX 坏了" | 从 main 开 `hotfix/xxx` → 修 → PR → merge main → tag `v0.x.y` → 部署 |
| **回滚** | "回到 v0.1.0" | `git checkout v0.1.0` + 重启服务 |
| **查版本** | "生产跑的什么" | `curl /api/version` 或指页脚 |

### AI 必须遵守的红线

1. **永远不把 feat/* 直接合进 main**
2. **部署前让负责人 review 改动清单**（列出这次发什么）
3. **hotfix 优先级 > 新功能**，负责人一说"生产坏了"就暂停手上的 feature
4. **CHANGELOG.md 每次发版必须更新**，记录"用户可感知的变化"

### CHANGELOG 格式

```markdown
## [Unreleased] — develop 未发布
见 git log main..develop

## [0.2.0] — 2026-05-15
### 新增
- XX 功能
### 修复
- YY bug
### 基建
- ZZ 重构（不影响用户）
```

## 发版流程脚本

```bash
# 发 0.2.0
git checkout develop && git pull
git checkout -b release/0.2
# E2E 回归测试
vim app/__version__.py    # __version__ = "0.2.0"
vim CHANGELOG.md          # [Unreleased] → [0.2.0]
git commit -am "release: v0.2.0"
git checkout main && git merge release/0.2
git tag -a v0.2.0 -m "release: 查重 + 监控 + ..."
git push origin main --tags
git checkout develop && git merge main && git push   # 同步

# 生产部署
ssh prod-host 'cd /app && git fetch --tags && git checkout v0.2.0 && docker compose restart app'
curl https://prod/api/version   # 验证 {"version":"0.2.0"}
```

## Hotfix 流程

```bash
git checkout -b hotfix/xxx main
# 修 bug
git commit -am "hotfix: ..."
git checkout main && git merge hotfix/xxx
git tag -a v0.1.1 -m "hotfix: xxx"
git push origin main --tags

# 部署
ssh prod-host 'cd /app && git pull && docker compose restart app'

# 关键：同步到 develop，否则下次发版 hotfix 改动会丢
git checkout develop && git merge main && git push
```

## 踩过的坑

### 坑 1：本地 main 堆积未测 feature → 紧急 hotfix 时卡壳

**症状**：生产 P0 bug 要修，但本地 main HEAD 领先 origin/main 28 个未审计 commits。`git push` 会带上所有 commits。

**错误做法**：`git push --force` 覆盖，上了未测代码。

**正确做法**：
```bash
# 从 origin/main 开 hotfix 分支（不是本地 main！）
git checkout -b hotfix/xxx origin/main
git cherry-pick <hotfix-commit>   # 只带 hotfix 的改动
git push origin hotfix/xxx
# 开 PR merge 到 origin/main
```

### 坑 2：`reset --hard origin/main` 看到的 HEAD 不对

**症状**：刚推了 hotfix 到 origin/main，本地 `git log origin/main` 显示的还是旧 HEAD。

**原因**：需要先 `git fetch origin` 才能看到远端最新状态。

### 坑 3：Docker 容器内没 `.git`，commit hash 显示 unknown

**解决**：Dockerfile 加 `ARG APP_COMMIT` + compose 传 `APP_COMMIT: ${APP_COMMIT:-}`；或者构建脚本把 `git rev-parse HEAD` 写到 image 里的 `/app/COMMIT` 文件。

### 坑 4：hotfix 合进 main 后忘记同步 develop

**症状**：下次发版时 develop → main 的 merge 覆盖掉 hotfix 改动，生产 bug 复活。

**防御**：hotfix 部署脚本末尾必须有 `git checkout develop && git merge main`。

## 不适用场景

- 多人协作：需要升级到完整 Git Flow（release/\* 管理多人测试）
- 持续部署（CD）：trunk-based development 更合适
- 有独立 staging 环境：main → staging → prod 的三环境流程

## 相关链接

- [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)
- [Semantic Versioning](https://semver.org/lang/zh-CN/)
- [Gitflow Workflow - Atlassian](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)

## 首次实战案例

2026-04-24 在 trade-ai 项目建立此工作流。当天就实战验证：
- 本地 main 积攒 28 个未审计 feature commits 时生产突发 Issue #99 翻版 bug
- 通过 `hotfix/confirmation-session-fix` 从 origin/main 开 + cherry-pick 单个 commit → 干净部署
- 同时把 28 个 commits 搬到 `develop`，main 回到 origin/main
- 打 v0.1.0 tag + 上线版本号 infra（`/api/version` + 页脚）

效果：生产恢复用时 5 分钟，未审计代码零污染。
