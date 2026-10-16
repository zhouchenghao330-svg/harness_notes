# Pi 组件 04：Context and Prompt

> 状态：已共同阅读并确认。本文重点记录消息如何进入模型 Context，以及 Pi 如何控制 Context 体积。

## 1. 核心结论

Pi 的 Context 不是一份静态历史，也不是单独的 Prompt 文件，而是多层处理的结果：

```text
资源文件 / 工具 / Session 历史
          ↓
ResourceLoader
          ↓
AgentSession 组装 Base System Prompt
          ↓
Agent.state.systemPrompt / messages / tools
          ↓ 每次模型调用前
transformContext()
          ↓
convertToLlm()
          ↓
Provider Context
```

最终发给 Provider 的 Context 主要包含：

```text
Context
├── systemPrompt
├── messages
└── tools
```

## 2. Tool Definition 和 Prompt 的双通道

工具通过两条通道进入模型请求：

```text
工具定义
  → Context.tools
  → Provider 的结构化工具参数
  → 模型可以生成 Tool Call

工具说明 / 使用规范
  → System Prompt
  → 模型看到的文字指导
```

`Context.tools` 中的工具定义包含名称、描述和参数 schema。工具的本地 `execute()` 函数不会发送给模型，只留在 Agent 进程中执行。

工具还可以提供：

- `promptSnippet`：加入 System Prompt 的工具简要说明
- `promptGuidelines`：加入 System Prompt 的工具使用规则

因此：

```text
Context.tools  = 模型能调用什么
System Prompt  = 模型应该什么时候、如何调用
```

## 3. 工具输出截断

工具输出截断发生在 Tool Result 进入消息历史之前，是 Pi 控制 Context 体积的第一道防线。

### 默认限制

共享截断工具默认使用：

```text
最多 2000 行
最多 50KB
```

长行还可能被单独截断，例如 grep 匹配行有自己的长度限制。

### 不同工具的保留方向

```text
read / find / grep / ls
  → 通常保留输出开头

bash
  → 通常保留输出末尾
  → 错误和最终结果经常位于末尾
```

### Bash 的完整输出保存

Bash 使用 `OutputAccumulator` 增量接收输出：

```text
持续接收 stdout / stderr
  ↓
只保留有界的显示内容
  ↓ 输出超过限制
完整输出写入临时文件
  ↓
Tool Result 返回截断预览 + fullOutputPath
```

模型看到的可能是：

```text
[Showing lines 1800-2000 of 12000.
 Full output: /tmp/pi-output-xxx.log]
```

这意味着：

- 模型 Context 不会被 300KB 或数 MB 的 Shell 输出直接撑爆。
- 完整结果仍然保留在临时文件中，后续可以通过路径再次读取。
- 这是输出截断，不是语义摘要压缩。

### 截断和 Context Transform 的区别

```text
工具截断
  → 工具结果第一次生成时就限制大小

transformContext
  → 消息已经存在后，在某次模型请求前调整可见消息
```

代码入口：

- `packages/coding-agent/src/core/tools/truncate.ts`
- `packages/coding-agent/src/core/tools/output-accumulator.ts`
- `packages/coding-agent/src/core/tools/bash.ts`

## 4. transformContext

`transformContext` 是 Agent Loop 在每次模型调用前提供的上下文调整钩子：

```text
AgentMessage[]
  ↓
transformContext()
  ↓
AgentMessage[]
  ↓
convertToLlm()
```

它可以用于：

- 删除旧消息
- 裁剪过大的消息
- 注入外部上下文
- 过滤 UI 专用消息
- 重排或替换消息

Pi 默认的 SDK wiring 是：

```text
transformContext()
  → ExtensionRunner.emitContext()
  → 依次调用 Extension 的 context handler
```

Extension Runner 会先对消息做 `structuredClone`。每个 Handler 可以返回新的 `messages`，后续 Handler 继续处理上一个 Handler 的结果；Handler 出错时记录 Extension 错误，但不会直接破坏整个模型请求。

因此需要区分：

```text
Agent.state.messages
  = Agent 的完整运行历史

transformContext() 返回值
  = 本次模型请求实际看到的消息视图
```

Pi 当前默认实现中，`transformContext` 主要是 Extension 的可插拔入口；核心 Compaction 并不是在这个钩子里完成的。

代码入口：

- `packages/agent/src/types.ts:180`
- `packages/agent/src/agent-loop.ts:288`
- `packages/coding-agent/src/core/extensions/runner.ts:984`
- `packages/coding-agent/src/core/sdk.ts:362`

## 5. convertToLlm 和消息组装

`convertToLlm()` 负责把 Pi 内部的 `AgentMessage` 转换成 Provider 能理解的标准消息。

```text
内部 AgentMessage
  ├── user / assistant / toolResult → 原样保留
  ├── custom → 转成 user message
  ├── bashExecution → 转成包含命令输出的 user message
  ├── branchSummary → 转成摘要 user message
  ├── compactionSummary → 转成摘要 user message
  └── excludeFromContext → 过滤
```

SDK 还会根据设置处理图片：如果启用 `blockImages`，user 和 toolResult 中的图片会被替换成文本占位符。

最终组装过程：

```text
AgentContext.messages
  ↓
transformContext()
  ↓
convertToLlm()
  ↓
llmMessages

{
  systemPrompt: AgentContext.systemPrompt,
  messages: llmMessages,
  tools: AgentContext.tools
}
  ↓
streamFunction()
```

这就是“内部历史”和“发送给模型的消息”之间的边界。

代码入口：

- `packages/coding-agent/src/core/messages.ts:148`
- `packages/agent/src/agent-loop.ts:294`
- `packages/agent/src/agent-loop.ts:297`

## 6. Session Context 选择

Session 文件中的所有 Entry 不一定都会进入模型 Context。

```text
Session Tree
├── message              → 通常参与 Context
├── custom entry         → 主要用于 UI / 状态，不参与 Context
├── custom_message       → 可以参与 Context
├── branch_summary       → 转成摘要消息
└── compaction           → 替代已压缩的旧历史
```

`SessionManager.buildSessionContext()` 会：

1. 从当前 leaf 找到当前分支路径。
2. 排除不参与 Context 的 Entry。
3. 处理 Branch Summary 和 Compaction Summary。
4. 只保留当前有效的消息路径。

因此恢复 Session 时不是简单读取整个 JSONL，而是：

```text
Session Tree
  → 当前 leaf 路径
  → compaction-aware entries
  → Agent.state.messages
```

代码入口：

- `packages/coding-agent/src/core/session-manager.ts:383`
- `packages/coding-agent/src/core/session-manager.ts:418`
- `packages/coding-agent/src/core/session-manager.ts:461`

## 7. Context Compaction

Compaction 是 Pi 的主要上下文压缩机制。

默认配置：

```text
reserveTokens: 16384
keepRecentTokens: 20000
```

当：

```text
当前 Context Tokens
  >
模型 Context Window - reserveTokens
```

就会触发压缩。

### 压缩流程

```text
当前 Session Context
  ↓
估算或读取 Context Token 数
  ↓
从最近消息向前寻找安全切点
  ↓
保留最近 keepRecentTokens
  ↓
把旧消息发送给摘要模型
  ↓
生成结构化 Summary
  ↓
写入 Compaction Entry
  ↓
旧历史替换为 Summary + 最近消息
```

### 切点规则

Pi 会尽量：

- 不在 Tool Result 中间切断。
- 保留完整的 Tool Call / Tool Result 关系。
- 如果切到了一个很大的 Turn 中间，额外生成 Turn Prefix Summary。
- 从旧 Compaction Summary 继续更新，而不是每次完全重写。
- 从历史中提取已经读取和修改过的文件，附加到摘要中。

### Token 估算

Pi 优先使用最近一次 Provider 返回的 usage；如果没有可用 usage，则使用消息估算：

```text
大约按字符数 / 4 估算 Token
图片使用额外的估算字符数
```

### Overflow Recovery

如果模型请求已经发生 Context Overflow：

```text
请求失败或响应被截断
  ↓
移除失败的 Assistant Message
  ↓
执行 Compaction
  ↓
Retry 一次
```

正常接近阈值时是预防性 Compaction；已经溢出时则是 Compaction + Retry。

代码入口：

- `packages/coding-agent/src/core/compaction/compaction.ts:235`
- `packages/coding-agent/src/core/compaction/compaction.ts:388`
- `packages/coding-agent/src/core/compaction/compaction.ts:751`
- `packages/coding-agent/src/core/agent-session.ts:2050`

## 8. System Prompt 组装结构

### 默认 Prompt 路径

当没有自定义 `SYSTEM.md` 时，结构大致是：

```text
默认角色定义
  ↓
Available tools
  ├── 当前工具的 promptSnippet
  └── 只展示有 snippet 的工具说明
  ↓
Guidelines
  ├── 当前工具的 promptGuidelines
  ├── 扩展提供的 guidelines
  ├── Be concise in your responses
  └── Show file paths clearly when working with files
  ↓
Pi 文档和 Examples 路径说明
  ↓
APPEND_SYSTEM.md
  ↓
<project_context>
  └── AGENTS.md / CLAUDE.md 等项目上下文
  ↓
Skills
  ↓
Current working directory
```

### 自定义 Prompt 路径

如果存在 `SYSTEM.md`，它替换默认基础 Prompt：

```text
SYSTEM.md
  ↓
APPEND_SYSTEM.md
  ↓
<project_context>
  ↓
Skills（read 工具可用时）
  ↓
Current working directory
```

自定义 Prompt 替换的是基础角色 Prompt，不会自动排除项目上下文、Skills 或工作目录。

### Prompt 什么时候重建？

Base System Prompt 会在这些情况重建：

- AgentSession 初始化运行时。
- Active Tools 改变。
- Extension 发现新的资源。
- ResourceLoader reload。
- Prompt 需要应用新的工具 snippets 或 guidelines。

如果 Extension 在 `before_agent_start` 返回新的 System Prompt，则当前运行使用临时 Override；下一次运行会重新回到 Base Prompt。

代码入口：

- `packages/coding-agent/src/core/system-prompt.ts:28`
- `packages/coding-agent/src/core/agent-session.ts:1034`
- `packages/coding-agent/src/core/resource-loader.ts:515`
- `packages/coding-agent/src/core/resource-loader.ts:1023`

## 9. 这一组件的设计判断

Pi 没有把所有上下文管理都塞进一个 `transformContext()` 函数，而是分层处理：

```text
单个工具输出过大
  → Tool Truncation

某类消息不应该发送
  → transformContext / convertToLlm

Session 分支或特殊 Entry 不应该进入
  → Session Context Builder

历史整体过长
  → Compaction

模型已经溢出
  → Overflow Recovery + Retry
```

这样每一层处理不同问题：

- 工具截断控制单次输出规模。
- Transform 控制单次请求的可见消息。
- Session Builder 控制历史路径。
- Compaction 控制长期对话规模。
- Overflow Recovery 处理已经发生的失败。
