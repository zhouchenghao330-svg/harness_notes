# Pi Harness 组件 07：Compaction and Branching

> 状态：已共同阅读并确认。

## 1. 核心结论

Pi 的 Compaction 和 Branching 不是简单的消息数组操作，而是由三个层次共同完成：

~~~text
AgentSession
├── 判断什么时候需要压缩
├── 处理手动压缩、阈值压缩和 Overflow Recovery
├── 调用 summarizer
└── 把新的 Context 写回 Agent

compaction/
├── 选择压缩边界
├── 生成结构化 summary
├── 处理 turn split
└── 追加文件操作元数据

SessionManager
├── 持久化 compaction entry / branch_summary entry
├── 维护 parentId 树
├── 选择当前 leaf 的路径
└── 从路径重建模型 Context
~~~

核心设计：

~~~text
完整 Session 历史不删除
        ↓
当前路径太长时生成 summary
        ↓
summary 作为新的 Session Entry 持久化
        ↓
buildSessionContext() 生成折叠后的 Context
        ↓
agent.state.messages 被替换
~~~

Compaction 解决“同一条路径太长”，Branch Summary 解决“切换路径后如何携带另一条路径的探索结果”。

## 2. 组件边界

主要代码：

~~~text
packages/coding-agent/src/core/agent-session.ts
  负责触发、运行时状态和 AgentState 更新

packages/coding-agent/src/core/compaction/compaction.ts
  负责 token 估算、切点选择、summary 生成

packages/coding-agent/src/core/compaction/branch-summarization.ts
  负责分支路径收集和 branch summary 生成

packages/coding-agent/src/core/compaction/utils.ts
  负责消息序列化、Tool Result 截断和文件操作元数据

packages/coding-agent/src/core/session-manager.ts
  负责 Entry 持久化、leaf、树遍历和 Context 重建
~~~

compaction.ts 中的核心函数基本是纯逻辑；SessionManager 负责把结果写入 Session；AgentSession 把二者接入实际运行时。

## 3. Compaction 的三种触发方式

### 3.1 手动压缩

入口通常是 /compact、RPC 或 Extension：

~~~text
abort 当前 Agent 操作
        ↓
读取当前 leaf 路径
        ↓
prepareCompaction()
        ↓
调用 summarizer
        ↓
appendCompaction()
        ↓
重新构建 agent.state.messages
~~~

手动压缩不会继续刚才被中断的 Agent turn。

### 3.2 阈值压缩

默认配置：

~~~text
enabled: true
reserveTokens: 16384
keepRecentTokens: 20000
~~~

触发条件：

~~~text
contextTokens > contextWindow - reserveTokens
~~~

例如模型 Context Window 为 128000，reserveTokens 为 16384，则大约在 111616 tokens 时触发。

reserveTokens 的作用是为后续模型请求预留空间，不只是 summary 的输出空间。

阈值检查主要发生在一次 Agent 响应结束后，也会在提交新 Prompt 前检查上一次响应，以覆盖中断响应或缺少有效 usage 的情况。

### 3.3 Overflow Recovery

如果 Provider 明确返回 Context Overflow，或者出现了一个异常短的 length 响应，Pi 会进行一次 compact-and-retry：

~~~text
Provider 返回 overflow / 可恢复 length
        ↓
从 AgentState 移除失败的 assistant message
        ↓
执行 Compaction
        ↓
agent.continue()
        ↓
重试当前 Agent turn
~~~

只允许进行一次 Overflow Recovery。第二次仍失败时停止，并报告恢复失败。

失败的 assistant message 可能已经在 message_end 时写进 JSONL Session，但它会从本次重试使用的 AgentState Context 中移除。

## 4. Compaction 如何选择保留范围

Pi 不直接按数组下标删除前半段，而是按照最近消息的估算 token 选择切点：

~~~text
从最新 Entry 向前遍历
        ↓
累计消息 token
        ↓
达到 keepRecentTokens
        ↓
向前寻找合法切点
        ↓
得到 firstKeptEntryId
~~~

合法切点包括：

- user message
- assistant message
- bash execution
- custom message
- branch / compaction summary

不能切在 Tool Result 上，因为 Tool Call 和 Tool Result 必须保持可解释的关系。

如果当前切点落在一个尚未完整结束的 turn 中，Pi 会识别出该 turn 的起点，把它拆成：

~~~text
旧历史
  ↓
turn prefix
  ↓
turn suffix
~~~

turn prefix 会单独生成摘要，turn suffix 作为最近内容保留。

### 估算方式

如果最近 assistant message 有有效 Provider usage，Pi 优先使用真实 usage，再估算其后的消息。

如果没有有效 usage，则使用字符数估算：

~~~text
估算 token ≈ 字符数 / 4
~~~

图片会使用固定的字符估算值。

## 5. Summary 的生成方式

Compaction 会额外发起一次 LLM 请求，而不是让当前 Agent Loop 继续生成：

~~~text
需要压缩的 AgentMessage
        ↓
convertToLlm()
        ↓
serializeConversation()
        ↓
放入 <conversation> 标签
        ↓
附加 summary instruction
        ↓
单独调用模型
~~~

这个 summarization 请求：

- 不携带工具；
- 设置 toolChoice 为 none；
- 使用独立的 summary system prompt；
- 对 transient stream failure 使用 retry；
- 如果模型返回 tool call、length 或 error，则不写入不完整 summary。

默认 summary 结构：

~~~text
## Goal
## Constraints & Preferences
## Progress
### Done
### In Progress
### Blocked
## Key Decisions
## Next Steps
## Critical Context
~~~

下一次 Compaction 会把之前的 summary 放进 previous-summary 标签，让模型更新原来的 checkpoint，而不是把多个完整 summary 无限堆叠。

### Split Turn 的摘要

如果切点落在一个 turn 中，Pi 会生成两个部分：

~~~text
历史 summary
    +
Turn Context summary
    +
保留的 turn suffix
~~~

Turn Context summary 重点保留：

- 当前 turn 的原始请求；
- prefix 中已经完成的工作；
- 理解 suffix 所需要的上下文。

## 6. 文件操作元数据：不是让模型负责记忆

Pi 会从 assistant 的 Tool Call 参数中提取文件操作，例如 read、write、edit 的 path。

这里需要区分两个时间点：

~~~text
prepareCompaction()
  ├── 提前提取 fileOps
  └── 准备需要总结的消息
          ↓
LLM 正常生成 summary
          ↓
computeFileLists(fileOps)
          ↓
formatFileOperations(...)
          ↓
summary += 文件操作标签
          ↓
持久化 CompactionEntry
~~~

因此，最终的 read-files 和 modified-files 不是让 LLM 生成的：

- fileOps 在 summary 请求前被确定性提取；
- LLM 先生成正常的结构化 summary；
- Harness 在 LLM 返回后追加文件列表；
- 最终 summary 才被写入 Session。

追加格式：

~~~text
<read-files>
path/to/read-only-file
</read-files>

<modified-files>
path/to/changed-file
</modified-files>
~~~

这样做的原因是：模型可能遗漏文件，或者把读取和修改混淆；由 Harness 从 Tool Call 中提取可以提供稳定的事实记录。

如果之前的 Compaction Entry 已经保存过文件操作，Pi 还会把之前的 details 合并进来，形成累计的文件追踪。

Branch Summary 使用相同的后处理策略。

需要和 Tool Result 截断区分：

~~~text
Tool Result 截断
  = 控制发给 summarizer 的输入长度

文件操作追加
  = 在 summary 生成后补充确定性元数据
~~~

Tool Result 在 summary 输入中最多保留约 2000 个字符；完整 Tool Result 不会因为这个动作从 Session 中删除。

## 7. Compaction 如何持久化和重建 Context

生成 summary 后，SessionManager 追加：

~~~text
CompactionEntry
├── summary
├── firstKeptEntryId
├── tokensBefore
├── usage
└── details
~~~

它是当前 leaf 的子节点：

~~~text
user1
  ↓
assistant1
  ↓
user2
  ↓
assistant2
  ↓
compaction(summary, firstKeptEntryId=user2)
  ↓
user3
  ↓
assistant3
~~~

完整 Entry 仍然在 JSONL 中。构建模型 Context 时，SessionManager 找到当前路径中最新的 Compaction Entry，然后生成：

~~~text
compactionSummary
user2
assistant2
user3
assistant3
~~~

旧的 user1 和 assistant1 被 summary 逻辑替代，但没有从 Session 文件中删除。

如果存在多次 Compaction，当前路径只使用最新的 Compaction 边界：

~~~text
旧 summary
    ↓
更新后的 summary
    ↓
最新保留消息
~~~

完成 Context 重建后：

~~~text
sessionManager.buildSessionContext()
        ↓
this.agent.state.messages = sessionContext.messages
~~~

这保证了压缩后的运行时状态和持久化 Session 保持一致。

## 8. Branching 和 Branch Summary

### 8.1 不带 Summary 的分支切换

不需要总结时，Pi 只移动 leaf：

~~~text
sessionManager.branch(targetId)
        ↓
当前 Context 改为 targetId 所在路径
~~~

旧分支不会删除，之后仍然可以再次导航回去。

如果目标节点是 user message，Pi 不直接把它当成已提交的新消息，而是：

- 把 leaf 放到该 user message 的 parent；
- 把原始文本返回给 editor；
- 允许用户修改后重新提交。

如果目标节点是 assistant、toolResult 或其他非 user 节点，则直接把目标节点作为新的 leaf。

### 8.2 带 Summary 的分支切换

用户从旧 leaf 切换到目标节点时，Pi 会：

~~~text
旧 leaf
    ↓
找到 old path 和 target path 的 common ancestor
    ↓
收集旧分支上独有的 Entries
    ↓
生成 branch summary
    ↓
把 summary 挂到目标路径
    ↓
重建 Agent Context
~~~

Branch Summary 的 summary prompt 与 Compaction 类似，也会记录：

- Goal
- Constraints & Preferences
- Progress
- Key Decisions
- Next Steps

它的目的不是压缩当前路径，而是告诉新分支：

~~~text
用户之前在另一条分支探索过什么
做过哪些决策
哪些方案失败
修改过哪些文件
接下来可以如何继续
~~~

### 8.3 为什么 Summary 挂在目标路径

Pi 把 branch_summary entry 挂到目标位置，而不是旧分支末尾：

~~~text
旧分支
  └── abandoned work

共同祖先
  └── branch_summary
      └── 新问题
~~~

原因是 Branch Summary 的消费者是目标分支。它不是旧分支的注释，而是新分支可以看到的上下文消息。

Entry 会保存 fromId，用来记录 summary 来源于哪一个旧 leaf。

## 9. Compaction 和 Branch Summary 的区别

| 维度 | Compaction | Branch Summary |
| --- | --- | --- |
| 触发原因 | 当前路径太长 | 从一条路径切换到另一条路径 |
| 作用对象 | 当前路径的旧历史 | 被放弃分支的独有历史 |
| 结果 Entry | compaction | branch_summary |
| 是否改变当前路径 | 折叠当前路径 | 在目标路径上增加上下文 |
| 是否删除旧历史 | 否 | 否 |
| 是否需要 LLM | 默认需要 | 用户选择总结时需要 |
| 主要目标 | 降低 Context 长度 | 避免切换分支后丢失探索结果 |

可以抽象成：

~~~text
Compaction
  同一条路径太长
      ↓
  旧历史 → summary

Branch Summary
  从路径 A 切到路径 B
      ↓
  路径 A 的探索 → summary
      ↓
  summary 挂到路径 B
~~~

## 10. 运行时取消、重试和 Extension 介入

Compaction 和 Branch Summary 都有独立的 AbortController：

~~~text
_compactionAbortController
_autoCompactionAbortController
_branchSummaryAbortController
~~~

Extension 可以在真正生成 summary 前：

- 取消操作；
- 提供自己的 compaction result；
- 提供自己的 branch summary；
- 修改 summary instructions；
- 追加 details。

默认的 summary 请求和普通 Agent 请求使用同一套 retry settings，但它们不进入普通 Agent Loop，也不会触发工具执行。

## 11. 设计取舍

### 优点

- 原始 Session 可恢复，压缩不会破坏历史；
- summary 是显式 Entry，可以审计和重建；
- Context 选择由 SessionManager 统一完成；
- 工具结果截断与文件操作追踪相互独立；
- 文件列表由 Harness 提取，不依赖模型记忆；
- Overflow Recovery 可以自动恢复一次；
- Branch Summary 让 Tree Navigation 不只是简单回滚。

### 代价和限制

- 每次 summary 都需要额外的 LLM 请求和 token 成本；
- summary 是有损压缩，模型只能看到摘要和保留窗口；
- 复杂 turn 可能需要两次 summary；
- Overflow Recovery 只能尝试一次；
- 旧历史仍在磁盘中，但不会自动重新进入当前 Context；
- Branch Summary 如果频繁使用，会给新分支增加额外上下文；
- summary 质量依赖 summarization model，但边界选择和文件追踪由 Harness 控制。

## 12. 关键代码证据

~~~text
packages/coding-agent/src/core/compaction/compaction.ts:126-136
  CompactionSettings 和默认配置

packages/coding-agent/src/core/compaction/compaction.ts:235-238
  阈值判断

packages/coding-agent/src/core/compaction/compaction.ts:403-460
  合法切点和 turn split

packages/coding-agent/src/core/compaction/compaction.ts:751-830
  Compaction preparation

packages/coding-agent/src/core/compaction/compaction.ts:859-965
  Summary 生成和文件操作追加

packages/coding-agent/src/core/compaction/utils.ts:88-149
  Tool Result 截断和 conversation 序列化

packages/coding-agent/src/core/compaction/branch-summarization.ts:108-145
  旧分支与 common ancestor 收集

packages/coding-agent/src/core/compaction/branch-summarization.ts:195-246
  Branch Summary 输入准备和 token budget

packages/coding-agent/src/core/compaction/branch-summarization.ts:293-379
  Branch Summary 生成和文件操作追加

packages/coding-agent/src/core/agent-session.ts:2030-2153
  自动触发、阈值检查和 Overflow Recovery

packages/coding-agent/src/core/agent-session.ts:2166-2348
  自动 Compaction 执行和 AgentState 重建

packages/coding-agent/src/core/agent-session.ts:3030-3220
  Tree Navigation 和 Branch Summary

packages/coding-agent/src/core/session-manager.ts:418-470
  Compaction-aware Context 重建

packages/coding-agent/src/core/session-manager.ts:1096-1119
  CompactionEntry 持久化

packages/coding-agent/src/core/session-manager.ts:1354-1405
  Branch 和 Branch Summary Entry
~~~
