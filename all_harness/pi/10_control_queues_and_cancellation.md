# Pi Harness：Control、Queues and Cancellation

> 状态：已共同阅读，Code-verified
>
> 范围：Agent 运行中的输入注入、Run/Turn 生命周期、队列消费、取消、优雅停止，以及 AgentSession 的外层控制。

## 1. 这一组件解决什么问题？

Agent 不只是“调用模型，得到回答”。运行过程中还需要处理：

- 用户在工具执行期间追加的新指令；
- 用户希望立刻影响下一次模型调用，还是等当前任务完成后再发送；
- 模型调用或工具执行被取消；
- 当前响应结束后，是否真的可以认为 Agent 已经空闲；
- 一次用户请求内部可能需要多轮模型和工具交互。

因此 Pi 在 Agent Loop 外围增加了一个运行控制层：

```text
Agent Loop
  负责：模型调用、工具执行、继续或停止

Control Layer
  负责：输入排队、运行生命周期、取消、等待空闲

AgentSession
  负责：把控制层接到 TUI、Session、Retry、Compaction 和 Extension
```

## 2. 三个容易混淆的概念

### 2.1 Model Request

一次对模型 Provider 的请求，也就是调用一次 `streamFunction`。

```text
Context → Model Provider → Assistant Message
```

它只是一次模型调用，不等于一个 Run。

### 2.2 Turn

在 Pi 的 Agent Core 中，一个 Turn 是：

```text
一次 assistant 响应
  + 这条响应触发的全部工具调用
  + 工具返回的全部 Tool Result
```

如果 assistant 没有发出工具调用，那么这个 Turn 就只有一次最终文本响应。

如果 assistant 一次发出三个工具调用，Pi 即使并行执行它们，它们仍然属于同一个 Turn。

### 2.3 Run

一个 Run 是从 `agent.prompt()` 或 `agent.continue()` 开始，到 `agent_end` 结束的完整 Agent 运行。

一个 Run 可以包含多个 Turn：

```text
Run
  ├── Turn 1：模型请求 → 搜索工具 → 搜索结果
  ├── Turn 2：模型请求 → 网页读取工具 → 网页内容
  ├── Turn 3：模型请求 → 补充检索工具 → 检索结果
  └── Turn 4：模型请求 → 最终文本回答
```

所以用户的一条复杂查询通常可以看成一个 Run，但不能把这条查询理解成只有一次模型请求。

## 3. 具体例子：调查菲尔兹奖最新新闻

假设用户输入：

```text
帮我调查一下有关菲尔兹奖的最新新闻。
```

一次正常的 Pi 运行可能是：

```text
用户消息进入上下文
  ↓
Run 开始
  ↓
Turn 1
  模型：调用 search("Fields Medal latest news")
  工具：返回搜索结果
  ↓
Turn 2
  模型：调用 search 进行补充检索
  工具：返回更多结果
  ↓
Turn 3
  模型：调用 fetch_page 读取重要文章
  工具：返回文章正文
  ↓
Turn 4
  模型：没有 tool call，整理并输出最终回答
  ↓
Run 结束，发出 agent_end
```

这正是 Harness 的核心价值：模型每次只做下一步决策，Harness 负责把工具结果放回上下文，再发起下一轮模型请求。

## 4. Run 和 Turn 在源码中的边界

`runAgentLoop()` 先启动一个 Agent Run，然后进入内部循环：

```text
agent_start
  ↓
turn_start
  ↓
调用模型
  ↓
处理 tool call 和 tool result
  ↓
turn_end
  ↓
还有工具链、steering 或 follow-up？
  ├─ 有：下一个 turn
  └─ 没有：agent_end
```

源码中，`turn_end` 发生在 assistant 响应和工具结果处理完之后；`agent_end` 发生在整个循环不再继续时。

因此：

```text
turn_end ≠ run 结束
agent_end = 当前 Agent Run 结束
```

## 5. 两个输入队列

底层 `Agent` 维护两个独立队列：

```text
Agent
  ├── steeringQueue
  └── followUpQueue
```

队列中的消息在真正被 Loop 消费前，不会进入 `state.messages`。

消息被消费时，Loop 才会发出 `message_start` / `message_end`，然后 Agent 才把它追加到当前上下文和运行消息列表中。

这意味着“排队”和“已经进入模型 Context”是两个不同状态。

## 6. Steering：影响正在进行的任务

### 6.1 语义

Steering 是：

> 当前任务还没结束，但用户希望影响下一次模型决策。

典型例子：

```text
模型正在执行：搜索新闻
用户输入：不要只看英文媒体，也查一下中文报道
```

这条消息进入 `steeringQueue`，不会强行打断当前搜索工具。

### 6.2 实际执行链路

```text
模型发出 tool call
  ↓
工具继续执行
  ↓
工具结果写入上下文
  ↓
当前 Turn 结束
  ↓
读取 steeringQueue
  ↓
追加 steering 用户消息
  ↓
下一次模型请求看到这条消息
```

所以 Steering 的关键点是：

- 不跳过当前 assistant 已经产生的工具调用；
- 不直接修改正在生成的模型响应；
- 在当前 Turn 完成后、下一次模型请求前注入；
- 注入后通常仍然属于同一个 Run。

### 6.3 代码行为

`steer()` 本身只做入队：

```ts
agent.steer(message)
```

Agent Loop 在当前 Turn 结束后调用 `getSteeringMessages()`，把取出的消息加入 `currentContext.messages`，然后开始下一个 Turn。

## 7. Follow-up：等当前任务自然完成后再处理

Follow-up 是：

> 当前任务先完整结束；如果 Agent 原本要停止，再启动一轮新的处理。

例如：

```text
当前任务：调查菲尔兹奖新闻
Follow-up：把结果整理成表格，并标出来源日期
```

它不会参与当前工具链的中途决策。

### 7.1 Follow-up 的检查时机

Agent Loop 只有在以下条件满足时才检查 Follow-up：

- 当前 assistant 没有继续的工具调用；
- 当前没有待处理的 steering 消息。

执行路径是：

```text
当前 Turn 结束
  ↓
没有更多工具调用
  ↓
没有 steering
  ↓
读取 followUpQueue
  ↓
有消息：继续下一个 Turn
无消息：发出 agent_end
```

正常情况下，follow-up 会在 `agent_end` 之前被消费。因此连续的 Follow-up 通常仍然属于同一个底层 Run。

如果 Extension 在 `agent_end` 事件处理器中又添加了队列消息，AgentSession 会在外层通过 `agent.continue()` 继续处理，这是会话层的补偿路径。

## 8. Queue Mode：一次消费一条还是全部消费

两个队列都支持：

### one-at-a-time

```text
队列：A、B、C

当前 Turn 结束 → 注入 A → 模型响应
下一个 Turn 结束 → 注入 B → 模型响应
再下一个 Turn → 注入 C
```

### all

```text
队列：A、B、C

当前 Turn 结束
  ↓
一次性注入 A、B、C
  ↓
发起一次模型请求
```

两种模式的差别是：

- `one-at-a-time`：每条消息都可以影响一次新的模型决策；
- `all`：把多条消息合并成同一轮模型请求的输入。

## 9. ActiveRun：整个 Run 的控制对象

Pi 的 `ActiveRun` 包含：

```ts
type ActiveRun = {
  promise: Promise<void>;
  resolve: () => void;
  abortController: AbortController;
};
```

它分别解决三个问题：

| 字段 | 作用 |
| --- | --- |
| `promise` | 表示整个 Run 何时完成 |
| `resolve` | 在 Run 和事件监听器结束后完成 `promise` |
| `abortController` | 向模型、工具和监听器广播取消信号 |

因此 `activeRun` 不是某一个 HTTP 请求的 Promise，而是整个 `agent.prompt()` / `agent.continue()` 运行的生命周期。

## 10. Agent 的运行状态转移

```text
idle
  ↓ prompt()
activeRun 创建
  ↓
isStreaming = true
  ↓
模型流式响应 / 工具执行 / 队列消费
  ↓
agent_end
  ↓
等待异步 listeners 完成
  ↓
清理 activeRun 和 streaming 状态
  ↓
idle
```

`agent_end` 只表示 Loop 不会再产生新的 Agent 事件。

真正的 idle 要等：

- `agent_end` 的异步监听器执行完；
- `streamingMessage` 清空；
- `pendingToolCalls` 清空；
- `activeRun` 被清理。

因此 `waitForIdle()` 等待的不是单纯的模型响应结束，而是完整 Run 的结算。

## 11. Abort：取消当前运行

调用：

```ts
agent.abort()
```

实际动作是：

```text
activeRun.abortController.abort()
  ↓
signal.aborted = true
  ↓
模型适配器、工具、监听器感知 signal
  ↓
当前运行返回 aborted 或抛出取消错误
  ↓
发出 turn_end / agent_end
  ↓
清理运行状态
```

`AbortController` 是协作式取消机制，不是强制终止机制。

它不能自动杀死：

- 不检查 `signal` 的工具实现；
- 忽略取消事件的网络请求；
- 正在执行的同步 CPU 代码。

所以 Harness 只能广播取消意图，具体组件必须实现取消响应。

## 12. Abort 和优雅停止的区别

### Abort

中止当前运行中的模型或工具工作：

```text
当前 Turn 可能还没完成
  ↓
发送取消信号
  ↓
尽快停止
```

### shouldStopAfterTurn

要求当前 Turn 正常完成，但不要开始下一个 Turn：

```text
当前 assistant 和工具全部完成
  ↓
shouldStopAfterTurn() = true
  ↓
不再读取 steering / follow-up
  ↓
agent_end
```

因此：

```text
Abort              = 中途取消
shouldStopAfterTurn = 在安全边界优雅停止
```

## 13. AgentSession 的外层控制

`AgentSession` 在底层 `Agent` 之上又维护了一层会话级运行状态：

```text
AgentSession._runAgentPrompt()
  ├── agent.prompt()
  ├── 自动重试？
  ├── 需要 Compaction？
  ├── Extension 是否新增队列消息？
  ├── agent.continue()
  └── agent_settled
```

因此需要区分：

- `Agent.state.isStreaming`：底层 Agent Run 是否正在执行；
- `AgentSession.isStreaming`：整个会话级流程是否还没结束，包括重试、compact 和 continuation。

交互层通常这样映射：

```text
普通 Enter       → streamingBehavior: "steer"
Alt+Enter        → streamingBehavior: "followUp"
Esc              → abort
```

AgentSession 还维护一份面向 UI 的文本队列，用于展示“哪些消息正在等待发送”。真正的 Agent 队列中存的是 `AgentMessage`；消息开始被消费时，Session 再从 UI 队列中移除对应文本。

## 14. 设计上的关键边界

### 队列状态不是历史状态

尚未消费的 steering/follow-up 不属于模型 Context，也不会作为已经完成的对话历史持久化。

只有消息真正被 Loop 消费并进入 `state.messages` 后，才成为当前运行上下文的一部分。

### Tool Result 是 Turn 的边界材料

工具结果不是独立的 Run，也不是独立的 Turn。它属于触发它的 assistant Turn，并成为下一次模型请求的输入。

### Run 是控制生命周期，不是业务问题

一个 Run 通常对应用户的一次任务，但它的边界由 Agent Loop 的事件生命周期决定，而不是由“用户发了几句话”决定。

如果 Follow-up 在当前 Run 结束前被消费，它仍属于同一个 Run；如果 Session 在 `agent_end` 之后因为 retry、compaction 或 Extension 队列再次 `continue()`，则会进入会话层的延续流程。

## 15. 源码证据

- `packages/agent/src/agent.ts`
  - `PendingMessageQueue`：队列和 `one-at-a-time` / `all` 消费模式
  - `ActiveRun`：运行 Promise 与 AbortController
  - `Agent.steer()` / `Agent.followUp()`：队列入口
  - `runWithLifecycle()` / `finishRun()`：运行生命周期
  - `processEvents()`：状态归约和异步监听器等待
- `packages/agent/src/agent-loop.ts`
  - `runLoop()`：Turn、工具调用、steering、follow-up 和 `agent_end`
- `packages/agent/src/types.ts`
  - `AgentLoopConfig`：`getSteeringMessages`、`getFollowUpMessages`、`shouldStopAfterTurn`
  - `AgentState`：`isStreaming`、`streamingMessage`、`pendingToolCalls`
- `packages/coding-agent/src/core/agent-session.ts`
  - `prompt()`：空闲输入和运行中输入的路由
  - `steer()` / `followUp()`：Session 层的消息扩展与入队
  - `_runAgentPrompt()`：Session 级 retry、compaction 和 continuation
  - `abort()` / `waitForIdle()`：会话级取消和等待空闲

## 16. 本章结论

```text
Model Request = 一次模型调用
Turn          = 一次模型调用 + 它触发的工具处理
Run           = 多个 Turn 组成的完整 Agent 执行

Steering      = 当前任务未结束时，影响下一次模型决策
Follow-up     = 当前任务自然完成后，再启动下一轮处理
ActiveRun     = 管理整个 Run 的 Promise 和 AbortController
Abort         = 协作式取消当前运行
```

Pi 的控制层本质上是在解决一个问题：

> 如何让 Agent 在不破坏当前运行一致性的前提下，接收新的控制输入，并且可以安全地继续、停止、取消或等待结算。
