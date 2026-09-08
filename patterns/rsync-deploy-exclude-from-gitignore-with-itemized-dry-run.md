# rsync 部署：排除规则直接读 .gitignore，部署前 itemize 干跑看 `deleting` 行

## 一句话结论

`rsync --delete` 部署时手写 `--exclude` 清单会漂：每漏一项，要么把本地垃圾传上去，要么把远端只此一份的文件删掉。**"生产 = main 的不变量"本来就意味着 gitignored 的东西不该上去**，所以让 rsync 直接以 `.gitignore` 为排除规则（`--filter=':- .gitignore'`），被排除的远端文件同时受保护不被 `--delete` 删；每次部署前 `-n --itemize-changes` 干跑，只看两类行：`*deleting` 和 `<f.s`（内容变了）。

## 场景

- 部署 = rsync 工作树到服务器 + 容器重建（不是 git pull）
- 服务器上有仓库之外、只此一份的东西：`.env`、手工备份 `.env.bak.*`、上传产物 `dist/`、token 文件
- 从 worktree 部署：worktree 是新检出，几百个文件的 mtime 都变了，干跑输出很吓人

## 详细说明

命令形态：

```bash
rsync -az --delete --filter=':- .gitignore' \
  --exclude='.git' --exclude='.env' --exclude='.smoke-token' \
  ./ root@host:/srv/app/
```

- `--filter=':- .gitignore'`：每个目录按其 `.gitignore` 生成排除规则（`:` = 逐目录合并，`-` = 排除）。**排除即保护**：远端有、本地被排除的文件，`--delete` 不会碰。
- `.git` / `.env` / token 这类不在 `.gitignore` 里的仍显式排除；`.env` 千万别让它进 gitignore 以外的任何通道。
- 干跑：`rsync -azn … --itemize-changes`，然后只看 `grep deleting`（会删远端什么）和 `grep -E '^<f\.s|^<f\+\+'`（内容/新增），`^<f\.\.t` 只是 mtime，从 worktree 部署时几百行都是它，忽略。

镜像 tag 顺手一条：同一天多次部署用 `2026-09-08b/c/d` 后缀，别覆盖已有 tag——回滚时每一张都在；运维 cron 固定用 `:current`，部署时 `docker tag <日期> current`，cron 才不会抱着旧镜像里的旧快照跑。

## 数据支撑

手写 exclude 清单在 2026-09-07～08 两天内三次误删远端文件，每次都是清单漏一项：

| 日期 | 漏的项 | 后果 |
|---|---|---|
| 09-07 | `.env.bak*`（只排了 `.env`） | 删掉远端一份手工备份 |
| 09-07 | `.claude/` | 本地 worktree 残留整目录上传到生产 |
| 09-08 | `dist/` | 删掉远端旧安装包副本 |

改成读 `.gitignore` 之后同日三次部署：干跑 `deleting` 行为零，内容变更行与提交 diff 逐文件对得上，473 行 mtime 噪声一眼过滤。

## 适用范围

- 适用于：rsync + 容器/进程重启形态的小规模部署；远端存在仓库外文件的任何场景
- 不适用于：远端目录本身就是构建产物目录（那里没有 `.gitignore` 语义）；用 git pull / 制品仓库部署的流程

## 相关

- [split-config-check-from-e2e-verification](split-config-check-from-e2e-verification.md)（部署后的冒烟同样要真打，不只回读配置）
- [verify-immediately-after-deploy-hits-stale-edge-state](../anti-patterns/verify-immediately-after-deploy-hits-stale-edge-state.md)
