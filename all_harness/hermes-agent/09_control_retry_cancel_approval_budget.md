# Hermes Agent 09：Control / Retry / Cancel / Approval / Budget

> 这一部分不是 Hermes 独有的 Agent 架构创新，而是围绕 Hermes 自身运行方式形成的一套工程控制面。它适合作为可靠性检查清单，而不是所有 Harness 都应该照搬的模块划分。

## 1. 核心结论

Agent Loop 负责让任务向前运行；Control Plane 负责回答五个问题：

~~~text
Budget       还允许开始下一步吗？
Retry        当前失败应该原样重试、修复后重试，还是直接终止？
Cancel       用户要求停止或修正时，正在运行的工作怎样退出？
Approval     这个动作是否取得了足够的用户授权？
Deadline     某个组件最多允许占用多长时间？
~~~

这些问题互相相关，但不是同一个维度：

~~~text
一次操作
├── 预算充足，但未经授权              → 不能执行
├── 已获授权，但 deadline 已到         → 应当停止等待
├── 可以重试，但用户已经 cancel        → 不应重试
└── 工具必须顺序执行，但不属于危险操作 → 不需要 approval
~~~

Hermes 的复杂度主要来自多入口、多 Provider、多工具环境、并行工具、子 Agent 和长会话。简单 Harness 通常不需要复制全部机制。

## 2. 总体控制流程

~~~text
                         一次用户 turn
                              │
                              ▼
                    外层 Agent iteration
                    ├── 检查 cancel
                    ├── 检查 iteration budget
                    └── 消耗一个逻辑 iteration
                              │
                              ▼
                      内层 API retry loop
                    ├── 请求格式修复
                    ├── credential refresh
                    ├── backoff retry
                    ├── provider fallback
                    └── context compression
                              │
                              ▼
                         模型响应
                    ┌─────────┴─────────┐
                 tool_calls             text
                    │                    │
             approval / guard       completion gates
                    │                    │
               tool deadline             ▼
                    │                  final
               tool results
                    │
               下一个 iteration
~~~

最重要的边界是：一次 Agent iteration、一次物理 Provider 请求和一次恢复动作不是同一个概念。

## 3. Retry 与 Iteration 分层

### 3.1 外层 iteration

外层循环表示一次真正的 Agent 推进：

~~~text
模型推理
  → 可能发出工具调用
  → 执行工具并回填结果
  → 再让模型观察结果
~~~

进入外层循环时，Hermes 才调用 `IterationBudget.consume()`。

### 3.2 内层 API retry

内层 retry 表示同一个逻辑请求的多次物理发送：

~~~text
同一个 logical request
├── attempt 1：503
├── attempt 2：network timeout
└── attempt 3：success

IterationBudget：消耗 1
物理 Provider 请求：实际发生 3 次
~~~

`agent.api_max_retries` 在当前实现中更接近 `max_attempts`：配置为 `1` 表示只尝试一次，并不发生重试。

### 3.3 TurnRetryState

`TurnRetryState` 是当前外层 iteration 的恢复登记表，而不是重试算法：

~~~text
TurnRetryState
├── Provider OAuth / credential refresh 是否尝试过
├── thinking signature 是否修复过
├── invalid encrypted content 是否清理过
├── image shrink 是否尝试过
├── multimodal tool content 是否降级过
├── native compaction rejection 是否处理过
├── 是否要用压缩后的 messages 重启
├── 是否要在 fallback Provider 上重启
├── 是否要进行 length continuation
└── 是否收到用户 correction，需要重建 request
~~~

每个恢复 flag 都是 one-shot guard，防止确定性错误重复触发同一修复，形成无限循环。`retry_count`、`max_retries` 和 `compression_attempts` 仍由主 loop 管理，因为它们是循环边界，不是恢复记录。

## 4. 错误分类与多重恢复上限

错误分类结果会携带：

~~~text
ClassifiedError
├── reason
├── retryable
├── should_compress
├── should_rotate_credential
└── should_fallback
~~~

典型映射：

| 错误 | 主要恢复方式 |
| --- | --- |
| 429、503、Provider overload | jittered backoff，必要时 fallback |
| 401、403 | refresh / rotate credential，然后 fallback 或 abort |
| context overflow、413 | compression 后重建请求 |
| invalid thinking signature | 清理不合法 replay 状态后重试一次 |
| 图片超过 Provider 限制 | shrink 后重试一次 |
| 明确的请求格式错误 | 针对性修复一次，否则 fail fast / fallback |
| 本地确定性 post-processing bug | 不进行无意义 Provider 重试 |
| empty / thinking-only / truncation | 进入各自独立且有界的 continuation ladder |

Backoff 加入 jitter，避免多个 session 同时遭遇限流后，又同时重新冲击 Provider。

Hermes 不使用一个全局 retry 数字管理所有恢复：

~~~text
普通 API retry               → api_max_retries
compression retry           → compression.max_attempts
length continuation         → 独立上限
thinking-only continuation  → 独立上限
empty response              → 独立且可按成本缩减的预算
dropped tool call           → 连续 stall 上限
verification / kanban gate  → 各自 continuation 上限
outer processing error      → 单 turn 小上限
~~~

Empty-response guard 还会识别连续 zero-output。Provider 已经确定性返回空内容时，会提前停止付费重放完整上下文。

## 5. 什么情况下 refund iteration

没有形成有效模型推进时，Hermes 会执行：

~~~text
api_call_count -= 1
iteration_budget.refund()
~~~

典型情况包括：

- 用户 correction 使当前请求作废；
- context compression 后必须重建请求；
- Provider 故障，切换 fallback 后重新发送；
- 请求尚未发出就发现本地 runtime context 不足；
- pre-API compression 占用了控制流程但没有调用 Provider。

这些情况是在重做同一个 logical iteration。

Length continuation、thinking-only continuation 和 empty retry 通常不 refund。Provider 已经进行了一次推理并消耗 token，只是结果不完整或不合格，因此应计入预算并受独立 ceiling 约束。

~~~text
没有产生有效模型步骤
    → refund，重做同一个 logical iteration

已经产生部分或异常模型输出
    → 不 refund，进入 bounded continuation
~~~

只有 `execute_code` 被调用的 RPC 式 iteration 也会退款，避免程序化工具调用快速耗尽普通 Agent 的循环预算。

## 6. Budget

### 6.1 IterationBudget

`IterationBudget` 是线程安全计数器：

~~~text
consume()   尝试占用一次 iteration
refund()    退回一次错误计数
used        已使用数量
remaining   剩余数量
~~~

父 Agent 和每个 subagent 拥有独立 budget，而不是共享全局池。因此多个 subagent 的总调用量可以超过父 Agent 自身的上限。这是 Hermes delegation 的资源模型，不是所有 Harness 都需要采用的设计。

### 6.2 耗尽后的交付

~~~text
iteration budget 耗尽
          │
          ├── 已存在 verification candidate
          │       └── 保留已经形成的回答
          │
          └── 没有可交付回答
                  └── 发起一次移除 tools 的总结请求
~~~

总结请求只能根据已有状态交代完成内容和剩余问题，不能继续调用工具。这避免预算耗尽表现成无响应退出。

### 6.3 Run budget 是软预算

`run_budget_seconds` 当前不是严格的 wall-clock kill switch。它主要：

1. 到达 80% 时，在最新 tool result 后附加 wrap-up notice，要求模型停止新增探索并开始交付；
2. 在 Provider stale timeout 使用默认值时，根据剩余时间向下调整，避免一次无响应请求吞掉大部分预算。

当前没有简单的 `elapsed >= budget → hard_stop`。它更像收尾提示与请求等待上限调节器，而不是绝对执行期限。

## 7. Cancel 是传播协议

~~~text
AIAgent.interrupt()
       │
       ├── 设置 agent._interrupt_requested
       ├── 标记 Agent 主执行线程
       ├── abort 当前 Provider HTTP / stream request
       ├── 标记正在运行的 tool worker threads
       ├── 递归 interrupt child agents
       └── retry wait / approval wait / tool loop 分别观察信号
~~~

Hermes 同时维护 Agent 级 bool 和 thread-scoped interrupt。Gateway 进程会承载多个并行 session；全局 Event 可能让停止 A 时错误终止 B、C。

### 7.1 Cooperative cancellation 与 abandon

合作工具主动检查 `is_interrupted()` 并自行退出。对于完全不检查信号的同步工具，Hermes 只能：

~~~text
发送 thread-scoped interrupt
    → 等待短暂 grace period
    → 仍未退出则停止等待
    → abandon daemon worker
~~~

Python 不能安全强杀任意线程。对子进程，Hermes 才能终止整棵 process tree，避免留下孤儿进程。

### 7.2 取消后补齐 tool result

已经声明的每个 `tool_call_id` 都必须获得结果，即使未执行：

~~~text
assistant(tool_call A, B, C)
tool(A, completed)
tool(B, cancelled)
tool(C, skipped)
~~~

这保证中断后依然满足 assistant/tool pairing，恢复会话时不会把非法协议发送给严格 Provider。

## 8. Redirect 与 Hard Cancel

~~~text
hard cancel
    用户要求停止
    → 终止当前 turn
    → 不再继续推理

redirect / correction
    用户在运行中修正任务
    → 中止过时请求
    → 保存必要 partial context
    → 加入 correction
    → refund 当前 iteration
    → 重建同一个 logical iteration
~~~

Hermes 会处理 response 与 correction 同时到达的竞态：只要 correction 在 active-request 边界内被接受，刚返回的旧 response 也会被视为 stale，不能继续执行其工具调用。

Hard cancel 与 compression commit 之间还有 commit fence，避免取消落在持久化边界中间，留下半完成的 compression 状态。

## 9. Approval 是原工具调用内的授权屏障

这里的 `dangerous` 与 05 中的 `unsafe/sequential` 不是同一概念：

~~~text
Tool scheduling unsafe / sequential
    讨论执行顺序和副作用依赖
    问题：能不能并行？

Approval dangerous
    讨论风险策略和用户授权
    问题：允不允许执行？
~~~

一个工具可以必须顺序执行但无需审批，也可以并行安全却被 plugin policy 要求审批。

~~~text
模型发出 tool_call(id=abc)
          │
          ▼
   security detection
          │
    ┌─────┴─────┐
  无风险        有风险
    │            │
 直接执行     等待用户决定
                 │
          ┌──────┴──────┐
       approve          deny / timeout
          │                 │
   执行原 tool_call      返回 BLOCKED
          │                 │
          └──────┬──────────┘
                 ▼
       tool result(id=abc)
~~~

现代 Gateway 有有效通知通道时，Agent thread 原地等待。Approval 不新开模型 turn；批准后仍执行原始 tool call，并沿用原 `tool_call_id`。

## 10. Approval 决策层级

~~~text
1. Hardline catastrophic block
   └── rm -rf /、mkfs、裸设备 dd、shutdown、fork bomb 等
       即使 yolo 也不能通过

2. approvals.deny
   └── 用户声明永远禁止，同样高于 yolo

3. yolo / approvals.mode=off
   └── 跳过普通审批，但不能跳过前两层

4. permanent / session allowlist
   └── 已授权风险类别直接通过

5. dangerous pattern detector + Tirith
   └── 合并 findings，只形成一个审批请求

6. smart approval
   ├── approve：只批准当前命令
   ├── deny：拦截；交互用户仍可做一次性 override
   └── escalate：交给用户判断

7. human approval
   ├── once
   ├── session
   ├── always
   └── deny
~~~

核心原则是 `Silence is not consent`。用户拒绝、等待超时、Gateway 通知失败，以及 plugin 要求审批但不存在人类交互通道时，都应 fail closed。

Cron 和 `-q` 不能即时询问用户，因此由 `approvals.cron_mode` 和 `approvals.single_query_mode` 明确决定。

### 10.1 Denial breaker

模型可能不断换说法尝试被 smart reviewer 拒绝的命令。Hermes 按 session 统计连续 smart-deny；达到阈值后，在 tool result 中加入 circuit-breaker 指令，要求停止变体重试并向用户报告。

它只改变当前 tool result，不修改 system prompt，因此不破坏 prompt cache。

### 10.2 并行审批合并

~~~text
worker A ─┐
worker B ─┼── 相同 command + 相同 policy keys
worker C ─┘
                     │
                     ▼
                一个 approval request
~~~

- `session` / `always` 可以被 follower 采用；
- `deny` / `timeout` 也会传播，避免重复骚扰用户；
- `once` 只授权 leader 那一次，其他调用不能借用。

## 11. Human Wait 与 Tool Deadline 分账

用户思考审批的时间不属于工具执行时间：

~~~text
tool deadline = 60 秒
用户审批等待 = 55 秒
工具实际运行 = 10 秒

如果不分账
  → 工具运行 5 秒后被错误判定 timeout
~~~

Hermes 用 `human_wait_window` 记录真正等待人类的区间，并从动态 tool deadline 中排除。

但 human wait 自身仍有约束：

- Approval 有独立 timeout；
- 可排除等待时间有 ceiling；
- 普通阻塞代码不能伪装成人类等待无限延期；
- 等待期间发送 heartbeat，避免 Gateway watchdog 误判为卡死。

通用原则：授权等待不属于工具执行耗时，但授权等待本身也必须有界。

## 12. Deadline

`agent/deadline.py` 试图统一三个问题。

### 12.1 Timeout resolution

~~~text
config.yaml timeouts.*
        ↓ 未配置
legacy env var
        ↓ 未配置
default
~~~

新行为配置应放进 `config.yaml`；env 只作为兼容桥。

### 12.2 Platform-safe clamp

- `None` 表示不设边界；
- 非正数归一化为 unbounded；
- 非数字配置被忽略，而不是令受保护路径崩溃；
- 极大 timeout 被钳制，避免平台 `time_t` overflow。

### 12.3 Deadline 责任归属

~~~text
Provider timeout
    Provider、SDK 或网络报告超时

Hermes DeadlineExpired
    Hermes 自己决定不再等待
~~~

两者必须区分，否则 Hermes 主动终止可能被错误分类成 Provider 故障，触发无意义的 credential rotation 或 fallback。

`run_bounded_async()` 超时后 cancel 并 abandon task，而不等待可能被 shield 的 cancellation；`run_bounded_sync()` 使用 daemon worker，超时后停止等待线程。如果 backend 可能被半完成操作污染，会将其标记为 suspect，要求复用前检查或重建。

外部 timer 可以帮助检测 event loop 阻塞并转储线程栈，但无法强行抢占正在执行同步调用的 Python event-loop thread。

当前 `deadline.py` 是迁移中的统一基础，并未覆盖所有 timeout。Tool executor 因为需要动态排除 human wait，仍保留自定义 deadline loop。

## 13. 与其他 Harness 的关系

一个简单 Harness 可能只需要：

~~~text
有限 API retry
+ 最大 agent steps
+ AbortSignal
+ 高风险工具确认
~~~

只有系统同时具有多 Provider、多 session、多执行线程、远程工具、人工审批、子 Agent 和恢复需求时，才会逐渐长出 Hermes 现在的控制面。

真正可迁移的是以下不变量：

1. Retry 必须有界，并区分确定性失败与瞬态失败；
2. Agent iteration 与物理 Provider attempt 必须分账；
3. Cancel 必须传播到 request、tool worker、subprocess 和 child agent；
4. 中断后仍要补齐 tool-call / tool-result 协议；
5. Approval 必须发生在副作用之前，silence 不能算 consent；
6. Human wait 与 execution time 应分账，但二者都必须 bounded；
7. Deadline 的来源和责任必须明确；
8. 预算耗尽后仍应向用户交付可理解结果。

## 14. 设计评价

值得学习的是控制不变量，而不是具体文件组织：

- API retry 与 Agent iteration 分层；
- 无效重建路径 refund logical budget；
- cancel 是跨请求、线程、工具、进程和子 Agent 的传播协议；
- approval 保留原 tool call 与 ID，不创建额外模型 turn；
- human approval wait 从工具执行 deadline 中剥离；
- 各退出路径维护消息配对和可恢复 transcript。

不必照搬的部分：

- `approval.py` 已接近六千行，是长期事故修复累积形成的 god-file；
- Retry bookkeeping 已抽出 `TurnRetryState`，但策略仍大量散布在超长 conversation loop；
- `deadline.py` 尚未覆盖所有历史 timeout；
- `run_budget_seconds` 是软预算，命名容易让人误以为是硬 deadline；
- 大量 Provider 和 surface 特例只服务 Hermes 当前产品形态。

因此这一部分更适合被记作：

> Hermes 针对自身多入口、多 Provider、多执行环境进行的控制面规整，以及一组可复用的可靠性不变量。

它不像 Memory、Context Engine 或 Agent Loop Repair 那样具有明显的独立架构创新。

## 15. 源码定位

| 主题 | 文件 |
| --- | --- |
| 主循环、retry/restart/cancel 检查 | `agent/conversation_loop.py` |
| IterationBudget | `agent/iteration_budget.py` |
| 单次 iteration 恢复状态 | `agent/turn_retry_state.py` |
| Jittered backoff | `agent/retry_utils.py` |
| 错误分类 | `agent/error_classifier.py` |
| Agent interrupt / redirect | `run_agent.py` |
| Thread-scoped interrupt | `tools/interrupt.py` |
| 工具 timeout 与 interrupt | `agent/tool_executor.py` |
| Approval policy / state / gateway wait | `tools/approval.py` |
| 通用 deadline 基础 | `agent/deadline.py` |
| Budget exhaustion finalization | `agent/turn_finalizer.py` |
| Iteration-limit summary | `agent/chat_completion_helpers.py` |
