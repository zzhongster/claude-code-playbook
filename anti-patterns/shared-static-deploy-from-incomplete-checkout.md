# 反模式：从一份不完整的本地目录部署共享静态站点——更新一个页面，删掉所有别人的

## 一句话结论

如果一个 Cloudflare Worker（或任何"本地目录整体同步到线上"的部署方式）同时托管好几个站点，本地哪份目录被部署，线上就会变成那份目录的样子。**同一个技能装在多处时，各处的 `public/` 内容可以差得很远**。从缺东西的那一份部署，线上就会少掉对应的站点，而且不会有任何报错。更新前先找准"上次是从哪份目录部署的"，只替换要改的那一个文件，部署前后对其余文件做哈希比对。

## 场景

- 私密分享站点 `s.trade-ai-sgp.com/h/<site>/` 托管在一个共用 Worker 上，所有站点的静态资源都随 `wrangler deploy` 整体同步
- 发布技能有好几份副本：`~/.claude/skills/…`、`~/.agents/skills/…`、仓库里的 `.agents/skills/…`
- 由 Claude Code 接手 Codex 或其他会话发布过的页面，要"在原链接上更新内容"

## 踩坑经历（2026-10-22）

接手 Codex 会话，要更新它 10-15 发布的华美私密报告。技能文档里写的路径是 `~/.Codex/…`，实际并不存在。顺手找到的 `~/.claude/skills/publish-preview/worker/public/h/` 里**只有 3 个旧站点，没有华美这个**。如果在这里替换文件后直接 `wrangler deploy`，线上其余 700 多个文件（若干客户私密报告）都会被删掉。

后来翻 Codex 的会话记录（找 `wrangler deploy` 的 `workdir`），才确定真正的目录是 `~/.agents/skills/publish-preview/worker`：里面有 761 个文件，华美页面的哈希和本地成品完全一致。

部署前还要再确认一件事：`wrangler deployments list` 显示最后一次部署就是 Codex 那次，中间没有别的会话从第三个目录部署过。

同一次接手还碰到第二件事：Codex 把工作目录放在 `/tmp/...`，**macOS 大约 3 天后把文件全部清掉，只剩空目录**。证据 JSON、截图、更新脚本都没了，后续更新只能以 `outputs/` 下的成品为准，再反推结构。

## 正确做法

1. **先找到部署源再动手。** 依据有两个：一是上一次部署命令的工作目录（翻会话记录、shell 历史）；二是"目标站点的线上文件在哪份目录里、哈希一致"。技能文档里写的路径只能当线索。
2. **只换一个文件，比对其余。** 先把除目标文件以外的所有文件做 `shasum` 并排序存下来，替换后再算一遍、`cmp` 两份。部署日志里应该只出现 "1 new or modified asset"。
3. **部署前看部署历史。** `wrangler deployments list` 的最后一条应该就是你知道的那次。如果中间有陌生的部署，先弄清楚它是从哪里发的，否则你的部署会把它的内容盖掉。
4. **部署后的验证：**
   - 不带 cookie 访问应返回 401；
   - 口令换 cookie 后访问应返回 200，且带 `charset=utf-8`；
   - 页面里应包含这次新加的内容。

   线上哈希和本地不一致是正常的：Cloudflare 会在页尾注入 challenge 脚本，用 diff 确认只差这一段就行。
5. **交付物和中间产物都别放 `/tmp`。** 证据、脚本、备份应该和成品放在同一个目录下（例如 `outputs/<项目>/{evidence,scripts,history}`）。

## 关联

- `anti-patterns/tmp-path-hides-rerun-pollution.md`：同样是 `/tmp`，那条讲的是重跑污染，这条讲的是被系统清空
- `anti-patterns/verify-immediately-after-deploy-hits-stale-edge-state.md`：部署后马上验证会命中边缘缓存的旧状态，用 `?cb=` 绕开
