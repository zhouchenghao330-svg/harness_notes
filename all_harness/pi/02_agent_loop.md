# Pi 组件 02：Agent Loop

> 状态：已共同阅读，结论以当前代码为准。Tool System 的深入分析另行进行。

## 1. 它解决什么问题？

Agent Loop 是 Pi 的低层执行控制器，负责推动这条运行链：

```text
模型响应
  ↓
判断是否有 Tool Call
  ↓
执行工具
  ↓
写入 Tool Result
  ↓
决定是否再次调用模型
```

它不负责：

- 业务逻辑
- System Prompt 的完整设计
- Session 持久化
- Provider 的 HTTP 实现
- Steering / Follow-up 队列的存储

这些能力由上层通过 `AgentContext`、`AgentLoopConfig` 和回调注入。

## 2. 输入和输出

核心入口是：

```text
runAgentLoop(prompts, context, config, emit, signal, streamFn)
```

输入：

- `prompts`：本次新加入的消息
- `context`：system prompt、历史消息、可用工具
- `config`：消息转换、Context 变换、队列读取、停止判断、工具 Hook
- `signal`：取消信号
- `streamFn`：模型流式调用函数

输出：

- 本次运行产生的新消息
- 一系列 AgentEvent：生命周期、流式响应、工具执行事件

`runAgentLoop()` 会复制当前消息数组，再把本次 Prompt 加进去，因此本次运行有自己的 Context 快照。

## 3. 运行结构

```text
runAgentLoop()
  │
  ├─ 添加本次 Prompt
  ├─ 发出 agent_start / turn_start
  │
  └─ runLoop()
       │
       ├─ 内层循环：Tool Call 和 Steering
       │    │
       │    ├─ 注入待处理消息
       │    ├─ 调用模型
       │    ├─ 得到文本或 Tool Call
       │    ├─ 执行 Tool Batch
       │    ├─ 写入 Tool Result
       │    └─ 决定是否进入下一轮
       │
       └─ 外层循环：Follow-up
            │
            ├─ Agent 本来已经准备停止
            ├─ 检查 Follow-up Queue
            └─ 没有 Follow-up 才发出 agent_end
```

## 4. 一次请求的执行轨迹

```text
用户消息
  ↓
streamAssistantResponse()
  ↓
transformContext()
  ↓
convertToLlm()
  ↓
构造 systemPrompt / messages / tools
  ↓
streamFunction()
  ↓
AssistantMessage
  ├─ 普通文本 → 当前 Turn 结束
  └─ Tool Call → 执行工具并生成 Tool Result
                         ↓
                    写回 Context
                         ↓
                    下一次模型调用
```

Agent 内部可以保留自定义 `AgentMessage`，只有在模型调用边界才通过 `convertToLlm()` 转成 Provider 能理解的消息。

## 5. Steering 和 Follow-up

### Steering

Steering 表示：当前运行还没有结束，但用户想尽快改变 Agent 的方向。

准确时机不是某一个工具结束后立刻插入，而是当前 Assistant Message 产生的整批 Tool Call 都处理完后：

```text
Assistant Tool Call
  ↓
执行全部 Tool Call
  ↓
生成并写入全部 Tool Result
  ↓
turn_end
  ↓
shouldStopAfterTurn 检查
  ↓
读取 Steering Queue
  ↓
把 Steering 消息加入 Context
  ↓
下一次模型调用
```

因此，如果同一个 Assistant Message 同时调用了 `search` 和 `read_file`，即使 `search` 完成时用户已经发送 Steering，也要等 `read_file` 也完成后再注入。

### Follow-up

Follow-up 表示：让当前运行先自然走到停止点，再追加一条消息。

```text
当前 Assistant Response 没有 Tool Call
  ↓
没有 Steering
  ↓
Agent 到达自然停止点
  ↓
读取 Follow-up Queue
  ├─ 有：加入 Context，继续调用模型
  └─ 无：agent_end，真正结束
```

这里的“Turn”指一次 Assistant Response 加上它产生的 Tool Call / Tool Result。没有 Tool Call 说明当前 Turn 完成，但不一定立刻发出 `agent_end`，因为还要检查 Follow-up。

在 Agent 层，运行中的新输入不能再次调用 `prompt()`，调用方需要明确选择：

```text
改变当前方向       → agent.steer(message)
当前任务完成后追加 → agent.followUp(message)
Agent 已经空闲     → agent.prompt(message)
```

## 6. Tool Call 的高层行为

本组件只记录 Loop 如何接收和处理工具结果，Tool System 的设计细节后续单独拆解。

- 默认允许同一条 Assistant Message 中的多个工具并行执行。
- 如果配置为 sequential，或某个工具声明必须 sequential，则按顺序执行。
- 工具完成事件可以按完成顺序发出，但 Tool Result Message 按模型原始调用顺序写回。
- 只有当一个 Tool Batch 中所有结果都标记 `terminate: true` 时，才跳过下一次模型调用。
- 如果模型输出因长度限制被截断，即使解析出了 Tool Call，也不会直接执行，而是生成错误 Tool Result，让模型重新发起完整调用。

## 7. Agent Loop 的边界

```text
AgentSession / Agent
  ├─ 提供 Context、Config、队列和事件接收器
  ↓
Agent Loop
  ├─ 推进模型 → 工具 → 结果的运行状态
  ├─ 组装每次模型调用的临时 Context
  └─ 发出结构化事件
  ↓
Model Adapter / Tool System / Session Manager
```

Agent Loop 是控制平面，不是完整 Harness。它决定“下一步做什么”，但不决定业务工具是什么、历史如何永久保存、System Prompt 如何生成。

## 8. 代码证据

- `packages/agent/src/agent-loop.ts:95`：`runAgentLoop`
- `packages/agent/src/agent-loop.ts:155`：双层循环 `runLoop`
- `packages/agent/src/agent-loop.ts:281`：模型调用和流式消息组装
- `packages/agent/src/agent-loop.ts:411`：Tool Batch 分派
- `packages/agent/src/types.ts:149`：`AgentLoopConfig`
- `packages/agent/src/agent.ts:282`：`steer()` 和 `followUp()` 队列入口

## 9. 暂不在本组件回答的问题

Codex 是否先通过 JavaScript 编排多个工具，再由 `exec` 执行，以及 Pi 是否采用同一种机制，放到 **Tool System** 组件中比较。
