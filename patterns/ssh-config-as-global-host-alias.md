# 把 `~/.ssh/config` 当作全局机器别名层

## 一句话结论

> 任何"怎么连到机器 X"的细节（IP / 用户 / 密钥 / 端口）都应该一次性沉淀到
> `~/.ssh/config`，项目里和 AI 会话里只引用别名 `ssh X` / `rsync X:` —
> 别在脚本和 prompt 里反复粘贴用户名 + IP + `-i ~/.ssh/xxx`。

## 场景

你在项目 A 里让 Claude 把构建产物推到家里的 mac mini（局域网 IP 192.168.8.54）。
典型卡点会按这个顺序出现：

1. Claude `ssh 192.168.8.54` → 失败：默认用了当前 macOS 用户名 `zhangzhong`
2. 你说"用户是 joey" → `ssh joey@192.168.8.54` 还是 Permission denied
3. 翻来翻去发现需要 `-i ~/.ssh/id_ed25519` + 公钥已经在对方 `authorized_keys`
4. 终于通了，结果三个月后你换到项目 B，同一台 mini 又要重新调一遍

更糟的是，这些信息往往散落在：
- 项目 A 的 `scripts/deploy-xxx.sh`（硬编码 IP）
- 某份 `docs/runbook.md`（写着 `ssh joey@xxx`）
- 你自己的脑子里（"哪个 key 对应哪台机器来着？"）
- 不同 Claude 会话各自摸索一次，每次都要花 5 分钟"探机器"

## 正确做法

### 1. 把机器信息提到 `~/.ssh/config`

```ssh-config
Host macmini mini
    HostName 192.168.8.54
    User joey
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
```

要点：

- **一行多别名**：`Host macmini mini` —— 长别名（语义清晰）和短别名（手敲快）同时给。
  项目脚本里用 `mini`，文档里用 `macmini`，都指向同一份配置。
- **`IdentitiesOnly yes` 是必须的**：不加的话，SSH 会把 `~/.ssh/` 下所有私钥
  按顺序试一遍。超过 5 个 key 时容易触发对方 `MaxAuthTries`，连"对的那把钥匙"
  都试不到就被踢。
- **`IdentityFile` 显式声明**：别依赖默认（id_ed25519 → id_rsa → ...）顺序，
  跨项目共享 config 时太脆弱。

### 2. 同一台机器，多条网络路径分别开 Host

```ssh-config
Host macmini mini            # 主路径：局域网 IP（在家时最快）
    HostName 192.168.8.54
    User joey
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host macmini-mdns            # 备用 1：mDNS（IP 变了也能找到）
    HostName JoeydeMac-mini.local
    User joey
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host macmini-ts              # 备用 2：Tailscale（不在家时走 VPN）
    HostName 100.119.3.46
    User joey
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

同一台机器三条路径都可用，按网络环境降级。脚本里固定写 `ssh mini`，环境出问题时
人手切到 `-ts` 或 `-mdns`，不用动脚本。

### 3. 项目里只用别名，不准出现 IP 和密钥路径

```bash
# ❌ 不要
ssh -i ~/.ssh/id_ed25519 joey@192.168.8.54 'cd ~/app && docker compose pull'
rsync -e "ssh -i ~/.ssh/id_ed25519" ./dist/ joey@192.168.8.54:~/app/

# ✅ 这样
ssh mini 'cd ~/app && docker compose pull'
rsync ./dist/ mini:~/app/
```

好处：

- 项目脚本零环境耦合，换机器只改 `~/.ssh/config`，所有项目一起跟着切
- Claude 在新会话里只需要知道"目标机器叫 mini"，不需要每次重新摸索 user/key
- `git grep "192.168"` 不会再扫到一堆硬编码 IP

## 给 Claude 用的提示词模板

下次让 Claude 部署/同步到某机器时，prompt 里只给别名：

> 把当前 docker compose 状态推到 `mini`，假设 `~/.ssh/config` 里已经配好别名。

Claude 会直接 `ssh mini ...`，不用兜圈子。如果别名没配，让 Claude 主动报错并
**写一段建议追加到 `~/.ssh/config` 的配置块**——而不是自作主张去拼 `ssh user@ip`。

## 反模式

| 反模式 | 后果 |
|--------|------|
| 部署脚本里硬编码 `joey@192.168.8.54` | IP 变了要全局搜替换；不同脚本不一致 |
| 把私钥拷贝到项目仓库 `deploy_key/` | 公私不分，泄露风险；每个项目都得管一份 |
| 别名 `Host *` 下塞业务机器 | 一改影响全部 host；丢失隔离性 |
| 不加 `IdentitiesOnly yes` | 多 key 时认证失败，错误信息还误导（看起来像 key 没配对） |
| 用别名但忘记 `User` 字段 | 不同 macOS 账号切换时连不上，每次都得加 `joey@` |

## 适用边界

- **单人多机器** 场景最适用（家里 mini / 公司 mac / VPS / NAS）
- 团队共享场景需要额外考虑：别名命名要团队统一，私钥不在 config 里直接出现
- 跳板机/复杂网络可用 `ProxyJump` 在 config 里一次性表达，依然让上层只看到别名

## 一次性沉淀的收益

实测：把 mini 的连接细节提到全局 ssh config 后，**同一个会话内**节省了
"探机器"的 3-4 轮往返；**跨会话**收益更大——任何项目的 Claude 第一次提到 mini，
直接 `ssh mini` 就通，不再重复试错。
