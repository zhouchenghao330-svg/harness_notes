# Pi 组件 03：Agent State and Lifecycle

> 状态：已共同阅读并确认。本文记录 Agent 运行时状态，不包含 Session 持久化状态。

## 1. 它解决什么问题？

`Agent Loop` 负责推进一次运行，`Agent` 负责持有当前运行时状态，并把 Loop 发出的事件归并到这些状态中。

它是一个有状态的 Agent 外壳，不是完整的产品化 Harness。

## 2. Agent 的运行时状态

```text
Agent
├── state.messages          已完成的消息历史
├── state.streamingMessage  当前正在流式生成的消息
├── state.pendingToolCalls  正在执行的工具
├── state.isStreaming       当前是否处于运行中
├── state.errorMessage      最近一次错误
├── steeringQueue           Steering 消息队列
├── followUpQueue           Follow-up 消息队列
└── activeRun
    ├── AbortController
    └── 当前运行 Promise
```

上图展示 Agent 的运行时核心结构。

完整的公开 `AgentState` 还包括模型运行所需的静态状态：

```text
AgentState
├── systemPrompt
├── model
├── thinkingLevel
├── tools
├── messages
├── isStreaming
├── streamingMessage
├── pendingToolCalls
└── errorMessage
```

需要区分：

- `AgentState` 是公开的 Agent 状态视图。
- `steeringQueue`、`followUpQueue`、`listeners`、`activeRun` 是 Agent 的内部控制状态。
- Session 文件、Compaction、Branch 等持久化状态不属于这里，由上层组件负责。

## 3. 消息状态

`messages` 只保存已经完成的消息。

流式生成过程中，消息经历以下状态：

```text
message_start
  ↓
streamingMessage = 当前部分消息
  ↓
message_update
  ↓
streamingMessage = 更新后的部分消息
  ↓
message_end
  ↓
streamingMessage 清空
消息正式追加到 state.messages
```

因此，UI 可以使用 `streamingMessage` 显示实时内容，而历史记录只读取 `messages`。

## 4. 工具执行状态

`pendingToolCalls` 是一个工具调用 ID 集合：

```text
tool_execution_start
  ↓
把 toolCallId 加入 pendingToolCalls
  ↓
工具执行中
  ↓
tool_execution_end
  ↓
从 pendingToolCalls 移除 toolCallId
```

它主要用于表示当前有哪些工具正在执行，以及让 UI 能够显示工具运行状态。实际的工具调度仍由 `Agent Loop` 完成。

## 5. Agent 生命周期

```text
Idle
  ↓ prompt() / continue()
创建 activeRun
  ↓
isStreaming = true
  ↓
调用 Agent Loop
  ↓
接收 AgentEvent 并更新状态
  ↓
agent_end
  ↓
等待所有事件监听器完成
  ↓
finishRun()
  ↓
isStreaming = false
  ↓
Idle
```

`agent_end` 发出时，Agent 不一定已经 Idle。Pi 还要等待所有 `agent_end` 监听器完成，之后才清理 `activeRun` 并恢复 Idle。

## 6. activeRun

`activeRun` 是当前运行的内部控制句柄：

```ts
type ActiveRun = {
  promise: Promise<void>;
  resolve: () => void;
  abortController: AbortController;
};
```

### AbortController

`agent.abort()` 调用 `abortController.abort()`，产生的 `signal` 会向下传递给：

- 模型流式请求
- 工具执行
- Tool Hook
- 其他当前运行逻辑

### Promise

`promise` 表示当前运行什么时候彻底结束。`waitForIdle()` 返回这个 Promise，因此它等待的不只是模型停止，还包括 `agent_end` 监听器完成。

### resolve

`resolve` 是内部结束这个 Promise 的函数，在 `finishRun()` 中调用。

### activeRun 的并发保护作用

只要 `activeRun` 存在，Agent 就认为自己正在运行：

```text
activeRun 存在
  ↓
Agent 正在运行
  ├─ 不能再次 prompt()
  ├─ 不能 continue()
  └─ 不能 reset()
```

因此：

- `isStreaming` 是对外暴露的运行状态。
- `activeRun` 是内部实现取消、等待和并发保护的控制对象。

## 7. Agent 如何启动 Agent Loop？

```text
prompt()
  ↓
检查是否已有 activeRun
  ↓
把字符串转换为 AgentMessage
  ↓
runWithLifecycle()
  ↓
runAgentLoop()
```

`prompt()` 不能在运行中再次调用。如果当前已有运行，调用方必须选择：

```text
改变当前方向       → agent.steer(message)
当前任务完成后追加 → agent.followUp(message)
Agent 已经空闲     → agent.prompt(message)
```

`continue()` 则用于从当前消息历史继续运行：

- 最后一条是 user 或 toolResult：调用 `runAgentLoopContinue()`。
- 最后一条是 assistant：优先处理排队的 Steering 或 Follow-up。
- 没有可继续内容时，不能直接从 assistant 消息继续。

## 8. Agent、Agent Loop 和 AgentSession 的边界

```text
AgentSession
  ├─ 产品化输入处理
  ├─ System Prompt 重建
  ├─ Session 持久化
  └─ Compaction / Retry / Extension
          ↓
Agent
  ├─ 持有 AgentState
  ├─ 管理生命周期和队列
  ├─ 创建 AbortController
  └─ 将事件归并到状态
          ↓
Agent Loop
  ├─ 调用模型
  ├─ 执行工具
  ├─ 写回临时 Context
  └─ 发出 AgentEvent
```

Agent 是中间的运行时层：它比裸的 Agent Loop 多了状态和生命周期，但还没有承担完整 Harness 的产品能力。

## 9. Reset 和错误处理

`reset()` 只能在 Agent 空闲时调用，清除：

- 消息历史
- 流式状态
- Pending Tool Calls
- 错误状态
- Steering / Follow-up 队列

它不会自动清除模型、工具或 System Prompt 等配置。

如果 Loop 抛出异常，`Agent` 会生成一条带有 `error` 或 `aborted` 状态的 Assistant Message，并补齐：

```text
message_start
message_end
turn_end
agent_end
```

这样上层监听器仍然可以收到一套完整的失败运行事件。

## 10. 代码证据

- `packages/agent/src/types.ts:337`：`AgentState`
- `packages/agent/src/agent.ts:61`：可变运行时状态结构
- `packages/agent/src/agent.ts:125`：`PendingMessageQueue`
- `packages/agent/src/agent.ts:161`：`ActiveRun`
- `packages/agent/src/agent.ts:173`：`Agent` 类
- `packages/agent/src/agent.ts:347`：`prompt()`
- `packages/agent/src/agent.ts:486`：`runWithLifecycle()`
- `packages/agent/src/agent.ts:546`：`processEvents()`
