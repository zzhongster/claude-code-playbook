# 反模式：`ThreadingHTTPServer` + `ctx.wrap_socket(httpd.socket)` 在 TLS 握手处串行化

## 一句话结论

用 Python 内置 `ThreadingHTTPServer` 起 HTTPS 服务时，**把 listen socket 整个 `wrap_socket`** 会让 TLS 握手发生在主 accept 循环里，多线程形同虚设——一个不发数据的客户端就能把全局 accept 卡死。

## 场景

写一个"小巧、自给自足"的 Python HTTPS 服务（订阅分发、内部 webhook、轻量 API 网关等），下意识照抄网上的写法：

```python
httpd = http.server.ThreadingHTTPServer((HOST, PORT), Handler)
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain(CERT, KEY)
httpd.socket = ctx.wrap_socket(httpd.socket, server_side=True)  # ⚠️ 这一行
httpd.serve_forever()
```

跑起来好像没问题，单元测试也过。直到某天员工说"订阅导入失败"。

## 踩坑经历

自建 VPN 机场，Python 写了个 `trade-ai-sub.py` 替换 3X-UI 内置订阅服务，监听 `:2096`，给 15 个员工分发 Clash 配置。

某天员工反馈 Clash 全部"导入失败"。诊断：

```
$ curl https://yicha-v.example.com:2096/clash/xxx --max-time 30
# TCP 三次握手成功，TLS Client Hello 发出后 30 秒无响应

$ ssh vps 'systemctl is-active trade-ai-sub'
active                            # 服务还活着

$ ssh vps 'ss -tlnp sport = :2096'
LISTEN 6  5  0.0.0.0:2096 …      # Recv-Q=6 堆了 6 个待 accept

$ ssh vps 'cat /proc/<pid>/stack'
tcp_recvmsg_locked               # 主线程卡在某个客户端的 read 上

$ ssh vps 'ss -tnp | grep python3'
ESTAB 0 0 vps:2096  164.52.24.184:41061   # 一个扫描器 IP
```

iptables / ufw / fail2ban 全无关。**Python 进程在跑，但被一个不发包的扫描器 IP 锁死了**。

## 根因

`socket.accept()` 在被 `wrap_socket` 处理过的 `SSLSocket` 上调用时，做的是：

1. 底层 `_sock.accept()` 拿到 client TCP 连接
2. **同步**调用 `do_handshake()` 完成 TLS

第 2 步是阻塞的。所以即使 `ThreadingHTTPServer` 准备给每个请求开一个子线程，那个子线程**永远拿不到 socket**——因为主线程还卡在 accept 出口的 TLS 握手上。

慢速连接攻击（Slowloris-style）天然破这种服务：

- 攻击者建立 TCP 但不发 TLS Client Hello
- `do_handshake()` 一直 `recv` 等数据
- 服务端 `recv` 在 Linux 内核里是无限阻塞的（没设 timeout）
- 主 accept 循环冻结，listen backlog 堆满 → 所有真实用户被拒之门外

这不需要恶意，公网每天都有大量扫描器（Censys、Shodan、Stretchoid、各种灰产）会半连半断地探一下你的 HTTPS 端口。运气好能撑几个月，运气不好开服几天就中招。

## 修复

让 TLS 握手发生在子线程里，并给 socket 加硬超时：

```python
SOCK_TIMEOUT = 10  # 给 TLS 握手 + 单次请求 I/O 的硬上限

class TLSThreadingHTTPServer(http.server.ThreadingHTTPServer):
    """让 TLS 握手发生在子线程，而不是主 accept 循环。"""

    def __init__(self, addr, handler, ssl_ctx):
        super().__init__(addr, handler)
        self.ssl_ctx = ssl_ctx

    def get_request(self):
        sock, addr = self.socket.accept()        # 只 accept TCP
        sock.settimeout(SOCK_TIMEOUT)            # 防止任何阶段无限阻塞
        ssock = self.ssl_ctx.wrap_socket(sock, server_side=True)  # 在子线程里握手
        return ssock, addr

    def handle_error(self, request, client_address):
        # TLS 失败 / timeout / 端口扫描会刷一堆 traceback，静默掉
        pass

def main():
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.load_cert_chain(CERT, KEY)
    httpd = TLSThreadingHTTPServer((HOST, PORT), Handler, ctx)
    httpd.serve_forever()
```

关键点：

- **listen socket 不要 wrap**，只在子线程的 `get_request()` 里 wrap 每个 client socket
- `sock.settimeout()` 是兜底——即便 TLS 握成功了，慢速发送请求体也照样会卡死那个子线程；超时让卡死的线程至少能死掉
- override `handle_error` 静默掉公网扫描器制造的握手错误，否则 journald 会被刷爆

## 验证方法

故障复现 + 修复验证用 `nc`：

```bash
# 开 3 个 TCP 连接不发任何数据
nc your.host 2096 < /dev/null &
nc your.host 2096 < /dev/null &
nc your.host 2096 < /dev/null &

# 同时发正常请求
curl https://your.host:2096/healthz --max-time 8
```

- **修复前**：curl 一直转圈直到超时
- **修复后**：curl 1 秒内 200，且 `SOCK_TIMEOUT` 秒后慢连接被服务端踢掉

## 适用范围

- **适用于**：Python 内置 `http.server.ThreadingHTTPServer` / `HTTPServer` 起的 HTTPS 服务
- **同理适用于**：`socketserver.ThreadingTCPServer` + `ssl.wrap_socket` 的任何组合
- **不适用于**：用 `gunicorn` / `uvicorn` / `aiohttp` / `nginx` 反代等真正异步或多进程方案的场景——它们各自有正确的 TLS 处理路径

## 题外话：为什么不直接上 nginx

加 nginx 反代当然能解决，但代价是：

- 多一个组件要维护、要更新、要写 reload hook
- 证书续期链路从 1 段变 2 段
- 这个服务本来就只服务十几个员工，QPS 不到 1，上 nginx 是杀鸡用牛刀

正确的修复就是 30 行 Python，不要因为踩了一个坑就给整个架构加层。

## 相关

- Python docs: [`socketserver` — Synchronous vs threading vs forking](https://docs.python.org/3/library/socketserver.html)
- 项目源：`vpn-node/scripts/trade-ai-sub.py`
- 故障时间：2026-05-12
