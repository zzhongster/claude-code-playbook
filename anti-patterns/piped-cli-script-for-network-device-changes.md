# 反模式：往网络设备 CLI 里用管道预先塞命令做配置变更（外加一条代理 TUN 造成的误判）

## 一句话结论

`{ echo cmd1; sleep 3; echo cmd2; ... } | ssh -tt 交换机` 这种写法，拿来跑只读巡检没问题，拿来改配置就不行：它看不到每条命令的返回。某条命令报 `Error:` 后面照样接着执行；设备多弹一个 `[Y/N]`，下一条命令就被当成了回答；密码输错一次，排好的命令就被整批吞掉。有一次半截执行，把 DHCP 地址池删了，却没建好新的。

## 场景

- 让 Claude Code 通过 SSH 操作华为、H3C 这类网络设备（交换机、路由器）。密码由用户输入，Claude 不经手
- 为了省去用户一行行粘贴，写一个 bash 脚本，把命令列表按固定间隔喂给 `ssh -tt`，再用 `tee` 把输出存下来
- 只读的 `display` 巡检用这种办法效果很好，于是顺手把**配置变更**也写成了同样的脚本

## 踩坑经历

2026-09-23，华为 S300-24T4S（V200R022）三层交换机兼做 DHCP 服务器。办公 WiFi 的 /24 地址池不够用，方案是给 Vlanif80 加一个从地址段，然后把 DHCP 从接口地址池改成全局地址池。华为官方 FAQ 说明，只有全局地址池支持按从 IP 分配地址。

脚本顺序是：先建全局地址池 `wifi-main`，把 `network 192.168.8.0 mask 255.255.255.0`、排除段和 4 条打印机静态绑定写进去；再建扩展地址池；最后在 Vlanif80 上执行 `dhcp select global`。

实际执行结果：

```
[S14-ip-pool-wifi-main]network 192.168.8.0 mask 255.255.255.0
Error: The subnet of the pool cannot be overlapped with that of other pools.
[S14-ip-pool-wifi-main]excluded-ip-address 192.168.8.2 192.168.8.20
Error: The IP address is not in the pool.
[S14-ip-pool-wifi-main]static-bind ip-address 192.168.8.21 mac-address ...
Error: The IP address is configured in another IP pool.
...
[S14-Vlanif80]dhcp select global
Warning: There are IP addresses allocated in the pool Vlanif80. Are you sure to delete the pool?[Y/N]:y
```

- **旧的接口地址池还在的时候，不能新建同网段的全局地址池。** 所以 `wifi-main` 没有网段、没有排除段，也没有静态绑定。
- 脚本不看返回，照样执行到 `dhcp select global`，预留的 `y` 正好回答了「删除旧地址池」。**旧地址池被删掉了，新地址池是空的**，结果所有新设备都拿到了扩展网段的地址，打印机的 IP 绑定也丢了。
- 同一个脚本里，路由器那一段的命令在用户输错一次密码时被全部吞掉。回显里只有登录横幅，**回程路由根本没有加上**，脚本却没有任何提示。

之后改由用户在交互式 SSH 里手动粘贴修复命令，Claude 用 `read_terminal` 读取输出核对，几分钟就修好了。好在当时是晚上，在线设备还在用旧租约，影响只有一部手机。

## 根因

1. **管道是单向的**：`stdin` 只管按时间往外送，不知道上一条命令是成功、报错，还是在等 `[Y/N]`。「出错就停下」根本无从实现。
2. **网络设备的确认提示是按情况出现的**：同一条命令，有时直接执行，有时要确认（比如 `reset ip pool ... conflict`、`dhcp select global`、`save`）。为了应付可能出现的提示而预埋 `y`，确认提示没出现时这个 `y` 就是一条无效命令，提示出现时它**会替你确认一个你没预料到的危险操作**。
3. **命令从 SSH 连接一建立就开始送出**：用户输错密码重试期间，一部分命令就已经送出并被丢弃了，或者在登录横幅还没刷完时就送到了。每次登录丢掉的命令可能都不一样。
4. **变更的前置条件只在设备上才能看出来**：接口地址池和全局地址池互斥、同网段不能重叠，这些限制在文档里分散在不同章节。事前的只读预检也检查不到，只有真正执行时才会报错。

## 替代方案

- **只读巡检**：继续用管道脚本（`display` 类命令）。结果存进文件，由 Claude 分析。
- **配置变更**：
  1. Claude 输出**分步命令块**，每一步写明「预期回显」和「出错就停下」。
  2. 用户在**已经登录的交互式 SSH 会话**里逐块粘贴，Claude 用 `read_terminal` 读取该标签页的输出，核对通过后再给下一块。
  3. 如果一定要自动化，就用 `expect` 或 Python 的 `pexpect`/`netmiko`：逐条发送命令，等到提示符后匹配回显，出现 `Error:` 就中止；遇到 `[Y/N]` 按白名单回答，不在白名单里的提示也中止。
- **有互斥关系的变更，顺序是「先拆旧的，再建新的」**，并且把这个窗口期写进变更方案。本例的正确顺序是：`dhcp select global`（删除旧的接口地址池）→ 马上配置 `wifi-main` 的网段和静态绑定，同时接受这中间会有几秒空窗。
- **改完不要马上 `save`**：先验证，确认无误再保存。改坏了只要重启设备就能回到原来的配置（本例保住了这条后路）。

## 同一次还踩到的一个坑：代理 TUN 模式把端口扫描变成「全部开放」

Mac 上开着 Clash 或 Surge 的 TUN（增强）模式时，发往非本机网段（比如 192.168.9.x）的流量会先进虚拟网卡 `utun`，网关是 `198.18.0.1`。代理会**在本地先接下 TCP 连接**，所以 `nc -z` 探测 22、23、80、443 端口全部显示「开放」，SSH 则表现为 `kex_exchange_identification: Connection closed by remote host`。这导致我误报了一次「交换机开着 Telnet」，SSH 连不上的原因也一度查偏了。

- 先用 `route -n get <目标IP>` 看流量走的是哪张网卡。显示 `interface: utun*` 时，所有探测结果都不可信。
- 探测时绑定物理网卡：`nc -z -G2 -b en0 <ip> <port>`、`curl --interface en0 ...`。这样才能测到真实的端口状态（本例交换机的 22 端口其实是关着的，真正原因是 STelnet 没开，而且源接口绑错了）。
- `dig`/`curl` 测外网时，DNS 也会被 TUN 劫持成 `198.18.x.x`，要换成直接访问 IP 来测试。

## 附：华为 S 系列 / AR 的几个配置细节

- **SSH 只开 STelnet 不够**，还要把「SSH 服务端源接口」指定为管理用的 VLANIF。否则 TCP 能连上，但设备会马上断开，客户端连 SSH 版本信息都收不到。在 AR 路由器上，二层口不能用作源接口。
- **全局地址池里没有 `ping packet 0` 这个写法**（报 `Too many parameters`）。关闭 ping 探测、减少无线终端造成误报冲突的命令，在接口地址池里是 `dhcp server ping packet 0`。全局地址池里的对应写法需要另外确认。
- 地址池里的 `Conflict` 地址并不是永远锁死：空闲地址和过期地址都分完后，设备会自动回收冲突地址（官方文档「DHCP 租期和地址池」有说明）。
- **晚上抓的地址池数据不能代表高峰**。当时 110 个已用加上 66 个已过期，说明最近一天实际有大约 176 个客户端来过；如果只看「空闲 124 个」，就会得出「地址够用」的错误结论。

## 适用范围

- 适用于：Claude Code 通过 SSH 或 Telnet 操作任何带交互式 CLI 的设备，包括网络设备、防火墙、存储设备、BMC，以及任何会弹确认提示的 REPL。
- 不适用于：有幂等 API 的设备（RESTCONF/NETCONF、厂商控制器），这类设备应该用 API 加事务或回滚来做变更。

## 相关

- [配置层确认 ≠ 端到端验证](../patterns/split-config-check-from-e2e-verification.md)
- [丢弃探针输出，把「无操作」读成了「有行为」](discarded-probe-output-reads-noop-as-behavior.md)
- [配置缺口伪装成上游故障](config-gap-masquerades-as-upstream-failure.md)
