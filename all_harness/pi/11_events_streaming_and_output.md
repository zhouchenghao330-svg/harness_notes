# Pi Harness：Events、Streaming and Output

> 状态：已共同阅读，Code-verified
>
> 范围：Provider 流式事件、Agent 事件、Session 事件，以及 TUI、RPC、Print 和持久化如何消费这些事件。

## 1. 核心结论

Pi 不让 Agent Core 直接操作 TUI，也不让每个 Provider 直接定义自己的 UI 行为。

它在中间建立统一的事件链：

~~~
Provider Adapter
  ↓
AssistantMessageEventStream
  ↓
Agent Loop
  ↓
AgentEvent
  ↓
AgentSessionEvent
  ├── TUI
  ├── RPC / JSON
  ├── Session 持久化
  └── Extension
~~~

同一个 Agent Core 因此可以支持：

- 实时交互式 TUI；
- 面向程序的 RPC / JSON 流；
- 只输出最终回答的 Print 模式；
- Extension 监听和修改运行过程；
- Session 消息持久化。

## 2. 三层事件协议

### 2.1 Provider 流事件

packages/ai 把不同模型 API 的 SSE、WebSocket 或 SDK 回调统一为：

~~~
start
text_start / text_delta / text_end
thinking_start / thinking_delta / thinking_end
toolcall_start / toolcall_delta / toolcall_end
done / error
~~~

这些事件回答：

> Provider 当前产生了什么模型输出？

### 2.2 Agent 运行事件

Agent Loop 不把 Provider 原始协议直接暴露给上层，而是转换成：

~~~
message_start
message_update
message_end

tool_execution_start
tool_execution_update
tool_execution_end

turn_start
turn_end
agent_start
agent_end
~~~

这些事件回答：

> Agent 当前处于哪个运行阶段？

### 2.3 AgentSession 事件

AgentSession 在 Agent 事件上增加会话级事件：

~~~
queue_update
compaction_start / compaction_end
auto_retry_start / auto_retry_end
agent_settled
entry_appended
session_info_changed
~~~

这些事件回答：

> 整个 Coding Agent 会话当前发生了什么？

## 3. Provider 流的基础设施

AssistantMessageEventStream 是一个异步事件流：

~~~
Provider Adapter
  ├── stream.push(event)
  └── stream.end()

Agent Loop
  └── for await (const event of stream)
~~~

它同时支持两种情况：

- Agent 正在等待事件时，事件直接交给消费者；
- Agent 暂时没有读取时，事件先进入内部队列。

流还维护一个 result Promise，在收到 done 或 error 后解析为最终 AssistantMessage。

因此模型流有两个出口：

~~~
中间过程 → AsyncIterator 逐个消费
最终结果 → result() 获取完整消息
~~~

## 4. 模型事件如何转换成 Agent 消息事件

streamAssistantResponse 是主要转换点。

### 4.1 开始生成

~~~
Provider: start(partialMessage)
  ↓
Agent: message_start
~~~

Agent 会把 partial message 放入当前运行上下文，供后续更新。

### 4.2 生成中

Provider 发出文本、thinking 或工具参数事件：

~~~
text_delta
thinking_delta
toolcall_delta
  ↓
Agent: message_update
~~~

Agent 会更新当前的 partial assistant message，并把最新快照发送给订阅者。

### 4.3 生成完成

~~~
done / error
  ↓
stream.result()
  ↓
Agent: message_end(finalMessage)
~~~

message_end 携带最终权威消息，之后 Agent Loop 才会判断是否存在 tool call 并开始工具执行。

## 5. 一次普通运行的事件顺序

以“调查菲尔兹奖最新新闻”为例：

~~~
agent_start
  ↓
turn_start
  ↓
message_start / message_end
  用户消息进入上下文
  ↓
message_start
  assistant 开始生成
  ↓
message_update*
  文本、thinking 或 tool call 参数持续更新
  ↓
message_end
  assistant 消息完成
  ↓
tool_execution_start
  ↓
tool_execution_update*
  搜索工具报告进度
  ↓
tool_execution_end
  ↓
message_start / message_end
  tool result 写入上下文
  ↓
turn_end
~~~

如果还需要继续调用模型：

~~~
turn_start
  ↓
下一次模型请求
  ↓
...
~~~

最终 assistant 不再发出工具调用：

~~~
message_end
  最终回答
  ↓
turn_end
  ↓
agent_end
  ↓
agent_settled
~~~

初始 prompt 的精确事件顺序是：Agent 先发出 agent_start 和 turn_start，然后发送 prompt 的 message_start / message_end，再开始 assistant 流式响应。

## 6. message_update 和 tool_execution_update 的区别

这是 Pi 事件模型中非常重要的边界。

### 模型正在生成工具调用

~~~
toolcall_start
toolcall_delta
toolcall_end
~~~

这些会被包装为 message_update，表示：

> 模型正在构造一个工具调用。

此时工具还没有执行。

### 工具真正开始运行

模型 assistant 消息结束后，Agent 才执行工具：

~~~
tool_execution_start
tool_execution_update
tool_execution_end
~~~

这些表示：

> 工具已经被 Harness 接受，当前正在执行或已经执行完成。

完整链路：

~~~
模型生成 tool call 参数
  → message_update

assistant 消息完成
  → message_end

工具开始执行
  → tool_execution_start
  → tool_execution_update
  → tool_execution_end

工具结果进入上下文
  → message_start / message_end(toolResult)
~~~

## 7. Partial Message 的设计

模型流中的 partial 是当前 assistant 消息的累积快照。

例如：

~~~
第一个事件：我先查询
第二个事件：我先查询最新新闻
第三个事件：我先查询最新新闻和来源
~~~

Agent 会不断更新当前临时消息：

~~~
context.messages 中的临时 assistant
  ↓
被下一份 partial 替换
  ↓
最终被 final AssistantMessage 替换
~~~

这样 TUI 不需要自己维护复杂的 delta 拼接逻辑，只需要显示最新快照。

在进程内的 AgentEvent 中，message_update 携带最新 assistant message；在 JSON 输出层，Pi 会去掉重复的 partial 快照，只保留必要的增量事件。

最终完整内容仍以 message_end 为准。

## 8. Agent 如何根据事件更新状态

Agent 的 processEvents 先更新内部状态，再调用 listeners：

~~~
事件到达
  ↓
Agent 更新 state
  ↓
通知 listeners
~~~

主要映射：

| 事件 | Agent 状态变化 |
| --- | --- |
| message_start | 设置 streamingMessage |
| message_update | 更新 streamingMessage |
| message_end | 清空 streamingMessage，追加到 messages |
| tool_execution_start | 将 tool call id 加入 pendingToolCalls |
| tool_execution_end | 从 pendingToolCalls 移除 id |
| turn_end | 记录 assistant 错误信息 |
| agent_end | 清理当前临时 streaming 状态 |

订阅者收到事件时，查询到的 Agent State 已经与该事件一致。

## 9. 事件监听和顺序

底层 Agent.subscribe 保存 listener 集合，并按照注册顺序依次等待：

~~~
事件
  ↓
Listener A 完成
  ↓
Listener B 完成
  ↓
事件处理完成
~~~

这带来两个结果：

- 事件处理顺序稳定；
- 异步 listener 可以形成自然的 backpressure。

agent_end 是最后一个 Agent 事件，但它的异步 listeners 仍然属于当前 Run。只有这些 listeners 完成后，Agent 才会清理 activeRun 并真正变为空闲。

## 10. AgentSession 如何转发事件

AgentSession 订阅底层 Agent：

~~~
AgentEvent
  ↓
AgentSession._handleAgentEvent()
  ├── Extension 先收到
  ├── Session listeners 收到
  ├── message_end 时持久化
  └── 增加会话级状态和事件
~~~

例如 message_end 的处理顺序中，Extension 可以先替换消息；之后 Session 才把最终消息交给普通 listeners 和 SessionManager。

Session 还会把底层 agent_end 扩展为带有 willRetry 信息的 Session 事件，并在重试、compact 和 continuation 全部结束后发出 agent_settled。

## 11. TUI 如何消费事件

TUI 是一个纯事件消费者。

### Agent 开始

收到 agent_start：

- 清空旧的 pending tool 显示；
- 打开 terminal progress；
- 显示 Working 状态。

### Assistant 流式输出

收到 message_start：

- 创建 AssistantMessageComponent；
- 把它加入聊天界面。

收到 message_update：

- 刷新文本和 thinking 内容；
- 发现 tool call 时创建或更新工具组件；
- 更新工具参数的临时状态。

收到 message_end：

- 使用最终消息刷新组件；
- 把流式组件标记为完成；
- 处理错误或取消状态。

### 工具执行

~~~
tool_execution_start
  → 创建或激活 ToolExecutionComponent

tool_execution_update
  → 更新工具进度和临时结果

tool_execution_end
  → 显示最终工具结果并移除 pending 状态
~~~

### Agent 结束

收到 agent_end：

- 关闭 Working 状态；
- 清理 streaming component；
- 清空 pending tools。

## 12. RPC / JSON 如何消费事件

RPC 模式不依赖 TUI，而是把事件转换为 JSON 行：

~~~
AgentSessionEvent
  ↓
toJsonEvent()
  ↓
serializeJsonLine()
  ↓
stdout
~~~

message_update 的 JSON 版本会移除重复的 partial 快照：

~~~
message_start → 初始消息
message_update → 增量事件
message_end → 最终完整消息
~~~

客户端可以据此构建自己的 UI 或自动化控制逻辑。

RPC 还会注册 backpressure listener，在 stdout 写入速度跟不上时暂停 Agent 事件处理，避免输出无限堆积。

## 13. Print 模式如何消费事件

Print 模式可以不显示中间事件：

~~~
session.prompt()
  ↓
等待整个 Session 流程完成
  ↓
读取最后一条 assistant message
  ↓
只输出最终文本
~~~

所以 TUI、RPC 和 Print 使用的是同一个 Agent Runtime，只是选择了不同的事件消费策略。

## 14. 事件与持久化的边界

Pi 不会把每个 message_update 都保存到 Session。

主要持久化边界是：

~~~
message_end
  ↓
SessionManager.appendMessage()
~~~

原因是：

- message_update 是临时流式状态；
- message_end 才是完整且稳定的消息；
- 保存每个 delta 会产生大量无意义的历史记录；
- 恢复 Session 时只需要最终消息。

事件层同时承担两个职责：

~~~
中间事件：驱动实时 UI 和状态显示
最终事件：确定历史和持久化边界
~~~

## 15. 为什么 Pi 的状态看起来很简单？

你的观察是对的：Pi 的公开 Agent State 确实很小：

~~~
systemPrompt
model
thinkingLevel
tools
messages
isStreaming
streamingMessage
pendingToolCalls
errorMessage
~~~

这不意味着 Harness 功能少，而是它把复杂性放在了清晰的边界中：

~~~
模型协议差异
  → packages/ai Provider Adapter

运行过程
  → Agent Loop

实时状态变化
  → AgentEvent

会话级功能
  → AgentSession

具体展示方式
  → TUI / RPC / Print
~~~

Agent State 只保留当前运行必须知道的事实，而不是为每个消费者复制一套状态。

## 16. 简单和高效来自哪里？

### 小状态面

Agent 不需要维护一个巨大的嵌套状态机。当前消息、流式消息、正在执行的工具和运行标志已经足够驱动核心行为。

### 事件驱动

状态只在明确事件到达时更新：

~~~
message_update → 更新流式消息
tool_execution_start → 增加 pending tool
tool_execution_end → 移除 pending tool
message_end → 写入完整消息
~~~

### 消费者解耦

TUI、RPC、Extension 和持久化都通过事件接入，不需要 Agent Core 了解它们的具体实现。

### 只在最终边界持久化

中间流不写 Session，减少 IO 和历史噪音。

### 流式和最终结果并存

需要实时显示的消费者使用事件流；只需要结果的消费者使用最终消息，不必重复实现模型调用逻辑。

## 17. 代价和限制

这种设计也有代价：

- 高频 message_update 会带来较多 UI 刷新；
- 不同消费者必须正确理解事件顺序；
- message_update 和 tool_execution_update 的边界需要额外学习；
- 取消、重试和 compact 会让 Session 层事件比 Agent Core 更复杂；
- 如果 Extension listener 阻塞，可能影响整个 Run 的结算；
- JSON/RPC 还需要额外处理序列化和 stdout backpressure。

所以 Pi 的“简单”不是把功能删掉，而是把复杂性分散到协议边界，并让核心状态保持最小。

## 18. 源码证据

- packages/ai/src/types.ts
  - AssistantMessageEvent：Provider 统一流事件协议
- packages/ai/src/utils/event-stream.ts
  - EventStream：异步迭代、事件缓存和最终结果 Promise
- packages/agent/src/agent-loop.ts
  - streamAssistantResponse：Provider 事件到 Agent 事件的转换
  - emitToolExecutionEnd / emitToolResultMessage：工具事件和结果消息
- packages/agent/src/agent.ts
  - processEvents：状态归约和 listener 顺序
- packages/agent/src/types.ts
  - AgentEvent：Agent 生命周期、消息和工具事件
- packages/coding-agent/src/core/agent-session.ts
  - _handleAgentEvent：Extension、监听、持久化和 Session 状态转发
- packages/coding-agent/src/modes/interactive/interactive-mode.ts
  - handleEvent：TUI 对不同事件的显示处理
- packages/coding-agent/src/modes/json-event.ts
  - toJsonEvent：流式事件的 JSON 传输格式
- packages/coding-agent/src/modes/print-mode.ts
  - Print 模式只读取最终 assistant message

## 19. 本章结论

~~~
Provider Stream
  = 模型正在产生什么

Agent Event
  = Agent 当前运行到哪一步

Session Event
  = 整个会话发生了什么

message_update
  = 模型输出还在生成

tool_execution_update
  = 工具已经开始执行

message_end
  = 消息完成，可以作为稳定结果

agent_end
  = Agent Loop 结束

agent_settled
  = Session 连同收尾逻辑全部结束
~~~

Pi 的高效之处可以概括为：

> 用很小的运行状态承载核心事实，用有序事件承载过程，用不同消费者承载展示、持久化和外部控制。

