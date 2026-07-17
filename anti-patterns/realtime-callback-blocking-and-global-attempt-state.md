# 反模式：实时回调同步干重活，重试状态跨 Attempt 复用

## 一句话结论

音视频实时回调只能复制 primitive 快照并尝试写入有界队列；编码、文件 I/O、阻塞锁和状态推进必须移到 worker。每次启动或重试还必须拥有独立的 readiness、成功写入确认和取消令牌，不能用全局回调数或全局写入进度判断当前 attempt 已经就绪。

## 场景

以下组合很容易同时踩中这两个坑：

- Core Audio、AVAudioEngine、ScreenCaptureKit、摄像头或其他实时采集 API；
- 启动后需要等待“收到稳定数据并成功写出”才进入 running；
- 启动失败会重建 graph/stream 并自动重试；
- 多个采集源共用 writer、progress tracker 或 health channel；
- Swift Concurrency、actor 隔离与系统提供的 realtime callback 混用。

表面症状通常很随机：同一设备前两次正常，第三次启动失败；测试全绿，实机偶发崩溃；短录正常，长录中偶发停流或音频缺口。

## 踩坑经历

2026-07-17，在一个 macOS 录屏软件的双音源漂移验证中，蓝牙麦克风使用 AVAudioEngine，系统声音使用 ScreenCaptureKit。初版连续短测出现 2 次 PASS、1 次 FAIL：失败 attempt 没收到自己的稳定缓冲，却被旧 attempt 延迟到达的全局 callback/progress 推进，日志一度呈现互相矛盾的“当前 attempt 未就绪，但全局计数已增长”。

进一步整分支审查又发现实时 callback 直接调用共享 `TraceWriter`。该调用在锁内创建 JSONEncoder 并同步 `FileHandle.write`，意味着音频 IO 线程可能等待另一条采集流、内存分配或磁盘。

同一轮还发生了一次只在硬件上出现的崩溃：把整个 capture lifecycle 标成 `@MainActor` 后，词法作用域内创建的 `AVAudioSinkNode` receiver 继承了 actor 隔离；Core Audio 在自己的 IO 线程调用它时触发 executor assertion，进程以 SIGTRAP 退出。普通单元测试没有调用真实音频 IO 线程，因此没有暴露问题。

## 根因一：实时线程越过了阻塞边界

危险代码通常长这样：

```swift
input.installTap(onBus: 0, bufferSize: 4_800, format: format) { buffer, time in
    metrics.record(buffer, time)       // blocking NSLock
    readiness.recordValidCallback()    // lock + continuation resume
    try writer.append(makeEvent(...))  // JSON encode + lock + FileHandle.write
    progress.markSuccessfullyWritten()
}
```

这里的问题不只是“磁盘可能慢”。Realtime callback 的预算是硬边界，任何一个动作都可能让它失去确定性：

- `NSLock.lock()` 可能产生优先级反转；
- JSONEncoder、字符串插值和错误本地化可能分配内存；
- 文件写入可能进入内核等待；
- continuation resume、health report 和日志会触发不可控调度；
- actor-isolated closure 被非 actor 的系统线程调用会直接触发运行时断言。

## 根因二：用全局进度回答 Attempt 局部问题

“系统一共收到过几个 callback”不能回答“当前 graph 是否已经稳定”；“全局成功写入数增长了”也不能证明“当前 attempt 已写出第一条数据”。

典型错误流程：

1. Attempt 1 超时，开始 teardown；
2. Attempt 1 的 callback 已越过 `shouldRecord` 检查，但尚未完成写入；
3. Attempt 2 创建，并记录全局 progress baseline；
4. Attempt 1 的延迟写入完成，推进全局 progress；
5. Attempt 2 误以为自己的第一条事件已成功写出，进入 running。

仅在取消时唤醒等待者还不够。已签发的旧写入许可也必须失效；否则 callback 在取消前通过检查、在取消后提交，仍会污染 raw trace 和 progress。

## 正确架构：两层边界

第一层是 realtime callback 边界：

```swift
nonisolated static func makeTapBlock(
    handoff: BoundedAsyncHandoff<CallbackPayload>,
    attempt: CaptureAttempt
) -> AVAudioNodeTapBlock {
    { buffer, time in
        let payload = CallbackPayload.copyingPrimitives(from: buffer, time: time)
        switch handoff.submit(.init(payload: payload, attempt: attempt)) {
        case .accepted:
            break
        case .full, .closed, .contended:
            failureReporter.reportConstantFailure()
        }
    }
}
```

Callback 侧只允许：

- 读取系统 buffer 的 primitive 字段；
- 构造不持有 buffer 生命周期的 `Sendable` 值快照；
- semaphore-now、try-lock 和 `queue.async` 组成的非等待提交；
- 队列满、关闭或竞争时 fail-closed。

第二层是 attempt 生命周期边界。专用串行 worker 按顺序处理：

1. 记录 metrics；
2. 推进当前 attempt 的 readiness；
3. 从当前 attempt 获取写入 admission token；
4. token 在 commit 时再次确认 attempt 仍 active；
5. 执行编码和文件写入；
6. 成功推进 progress 后，才确认当前 attempt 的 written-event gate。

取消与提交必须线性化：

- cancellation 先赢：旧 token 永远不能提交；
- commit 先赢：cancel 等待该次提交结束，返回后不再有旧提交；
- 写入确认已经开始等待后若超时，直接 fail-closed，不再自动 retry。

Realtime callback 使用非阻塞 admission；worker 已离开 realtime 线程，可以使用阻塞、线性化的 admission。不要把 worker 与 MainActor 正常争锁误判为硬件故障。

## 停止顺序

可靠停止顺序是：

1. 标记 stopping，取消当前 attempt 和 admission；
2. 移除 tap/output，停止 engine/stream，阻止新 callback；
3. 关闭 handoff，拒绝新任务；
4. 同步 drain 所有已接受任务；
5. 最后关闭 writer 并发布汇总。

`closeAndFlush()` 与 `submit()` 必须在线性化锁内确定先后，避免“任务已占容量但尚未 enqueue，flush barrier 反而先进入队列”的超车竞态。故障 reporter 也要先在锁内 enqueue，再允许 close 插入 drain barrier。

## 数据支撑

本次修复使用真实蓝牙设备 EDIFIER W830NB（16 kHz 单声道）验证：

| 阶段 | 结果 |
|---|---|
| 修复前连续短测 | 2 PASS / 1 FAIL，失败由旧 attempt 的全局 callback/progress 污染触发 |
| Attempt-scoped readiness/ack 后首次硬件测试 | Audio IO 线程因 actor-isolated sink callback 触发 SIGTRAP |
| 最终 3 次短测 | 3/3 PASS，均 attempt 1 启动，无崩溃、健康故障、队列拒绝或 flush 错误 |
| 60 分钟实测 | 系统音频 180,259 事件；麦克风 12,018 事件；两路各覆盖约 3,605 秒 |
| 漂移门禁 | 相对时钟估计 0 ppm；两路 bounded residual 0 ms；最大 timestamp discontinuity 0 ms；PASS |
| 自动化验证 | 98/98 strict-concurrency + warnings-as-errors 测试通过 |

TDD 中分别取得了有效 RED：队列满仍错误接受、flush 提前返回、取消后旧 token 仍提交、旧 attempt 写入误确认新 attempt，以及 activation/admission 正常争锁被误报为故障。每条 RED 都在修复前指向对应竞态，而不是只做源码形状断言。

## 适用范围

- 适用于：音视频采集、传感器、交易行情、串口、网络包等有回调时限且支持自动重试的系统。
- 适用于：一个或多个 producer 共享 writer/progress，但 readiness 与 ack 必须按 session/attempt 隔离的系统。
- 不适用于：普通业务线程上没有实时性要求、没有自动 retry、允许 producer 阻塞等待的批处理。
- 队列容量不能照抄 64/256；应根据 callback 频率、允许突发时长和内存上限计算，并对 overflow 明确 fail-closed 或可量化丢弃策略。

## 相关

- [TDD 假 RED](tdd-fake-red.md)：并发回归测试也必须证明旧实现会在正确路径上失败。
- [整分支终审抓涌现缺陷](../patterns/whole-branch-review-catches-emergent-defects.md)：实时写盘与跨 attempt 污染都是分任务局部评审未发现、整体验收才暴露的问题。
