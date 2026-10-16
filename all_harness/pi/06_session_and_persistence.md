# Pi Harness 组件 06：Session and Persistence

> 状态：已共同阅读并确认。

## 1. 核心结论

Pi 的 Session 不是 AgentState 的持久化副本，而是一份可以重新构建 AgentState 的长期记录。

~~~text
AgentState
├── 当前 messages
├── systemPrompt
├── tools
├── steering / follow-up queues
├── activeRun
└── AbortController
        ↓
      运行时状态，不持久化

SessionManager
├── JSONL 文件
├── message entries
├── model / thinking 变更
├── compaction / branch summary
├── extension custom entries
└── parentId 树结构
        ↓
      长期状态，可恢复
~~~

恢复 Session 时，Pi 会重新创建 Agent、Prompt、Tools、Extension Runtime 和控制状态，只从 Session 中恢复必要的历史与配置状态。

## 2. 组件边界

~~~text
Agent Core
  负责运行 Agent、维护临时状态、发出事件
        ↓
AgentSession
  监听 Agent 事件，决定哪些事件要持久化
        ↓
SessionManager
  管理 JSONL、Entry、leaf、树遍历、恢复和分支
        ↓
Session File
  保存长期历史
~~~

因此：

~~~text
AgentState          = 当前运行时视图
SessionManager      = 可恢复的历史来源
buildSessionContext = 从历史构建模型 Context 的投影过程
~~~

AgentSession 在创建时会订阅 Agent 事件，并同时负责 Session 持久化、扩展事件、自动压缩和重试逻辑。

主要代码：

~~~text
packages/coding-agent/src/core/agent-session.ts:382
packages/coding-agent/src/core/session-manager.ts:845
~~~

## 3. 存储格式：JSONL

Pi 使用 JSONL 文件保存 Session，每一行是一条独立记录：

~~~text
Session Header
      ↓
Entry 1
      ↓
Entry 2
      ↓
Entry 3
~~~

Session Header 记录：

~~~text
type: session
version
id
timestamp
cwd
parentSession?
~~~

普通 Entry 通常包含：

~~~text
type
id
parentId
timestamp
~~~

默认目录是：

~~~text
~/.pi/agent/sessions/<encoded-cwd>/
~~~

文件名通常包含创建时间和 Session ID。

逻辑上，Session 是 append-only 的：

- 新 Entry 追加到文件；
- 旧 Entry 不被修改或删除；
- 分支通过 parentId 形成；
- 只有版本迁移或提取新分支时会重写/创建文件。

## 4. Session Entry 类型

| 类型 | 参与模型 Context | 作用 |
| --- | --- | --- |
| message | 是 | user、assistant、toolResult 消息 |
| model_change | 否 | 记录模型切换 |
| thinking_level_change | 否 | 记录 thinking level 切换 |
| compaction | 是 | 保存压缩摘要 |
| branch_summary | 是 | 保存被离开分支的摘要 |
| custom | 否 | 扩展自己的持久化状态 |
| custom_message | 是 | 扩展注入 Context 的消息 |
| label | 否 | 给节点添加或清除标签 |
| session_info | 否 | Session 名称等元数据 |

### custom 和 custom_message

这是扩展持久化中的重要区别：

~~~text
custom
  = 扩展状态
  = 保存到 Session
  = buildSessionContext() 时忽略

custom_message
  = 扩展消息
  = 保存到 Session
  = 参与 Agent Context
  = 后续由消息转换层决定如何交给模型
~~~

## 5. 一次对话如何被持久化

~~~text
用户输入
  ↓
Agent Loop 发出 message_end(user)
  ↓
AgentSession.appendMessage(user)

模型响应
  ↓
message_end(assistant)
  ↓
AgentSession.appendMessage(assistant)

工具执行完成
  ↓
message_end(toolResult)
  ↓
AgentSession.appendMessage(toolResult)

下一轮模型响应
  ↓
继续追加 assistant / toolResult
~~~

Pi 只在 message_end 时保存最终消息：

~~~text
message_start   → UI / listener
message_update  → UI / streaming
message_end     → 持久化
~~~

流式生成中的 partial assistant message 不会逐片写入 Session 文件。

AgentSession 的内部处理顺序是：

~~~text
Agent Event
    ↓
扩展事件处理
    ↓
Session listeners
    ↓
SessionManager.appendXXX()
~~~

### 新 Session 的首次写入

新 Session 在只有 user 消息时，可能还不会立即创建有效的磁盘文件：

~~~text
首次 user message
  → 先加入内存中的 fileEntries
  → 暂不 flush

首次 assistant message
  → 一次性写入 header、user 和 assistant
  → 后续 Entry 继续 append
~~~

这样可以避免用户刚输入就退出时留下大量没有回答的空 Session 文件。

相关代码：

~~~text
packages/coding-agent/src/core/agent-session.ts:620
packages/coding-agent/src/core/session-manager.ts:1015
~~~

## 6. AgentState 和 Session 的关系

两者不是同一份数据：

| AgentState | Session |
| --- | --- |
| 当前运行时的 messages | 所有分支的历史 Entries |
| 当前 systemPrompt | 不保存完整 Prompt |
| 当前 active tools | 不保存完整 Tool 实例 |
| queues | 不持久化 |
| activeRun | 不持久化 |
| AbortController | 不持久化 |
| streamingMessage | 不持久化 |
| errorMessage | 不作为主要历史恢复 |

Session 主要保存：

~~~text
messages
model
thinking level
compaction / branch summaries
extension state
session metadata
~~~

因此，Session 恢复不是“把旧 Agent 冻结后继续”，而是：

~~~text
读取历史
  ↓
创建一个新的 Agent Runtime
  ↓
恢复 messages 和必要配置
  ↓
重新加载 Prompt、Tools、Extensions、Auth
  ↓
继续运行
~~~

## 7. Session 恢复流程

~~~text
SessionManager.open()
        ↓
loadEntriesFromFile()
        ↓
解析 JSONL
        ↓
校验 Session Header
        ↓
迁移旧版本格式
        ↓
建立 byId、labelsById、leafId
        ↓
buildSessionContext()
        ↓
得到当前分支的 messages、model、thinkingLevel
        ↓
createAgentSession()
        ↓
恢复 agent.state.messages
        ↓
创建新的 AgentSession Runtime
~~~

启动时：

- 如果调用者没有显式指定模型，Pi 尝试从 Session 恢复模型；
- 如果模型不存在或认证不可用，则选择备用模型；
- thinking level 从当前分支的 thinking_level_change 恢复；
- 新 Session 会保存初始 model_change 和 thinking_level_change。

相关代码：

~~~text
packages/coding-agent/src/core/sdk.ts:191
packages/coding-agent/src/core/sdk.ts:374
~~~

## 8. Session 是一棵消息树

Session Entry 通过 parentId 连接：

~~~text
root
  ↓
user-1
  ↓
assistant-1
  ├── user-branch-A
  │     ↓
  │   assistant-A
  │
  └── user-branch-B
        ↓
      assistant-B
~~~

SessionManager 中的几个视图：

~~~text
getEntries()
  = 当前文件中的全部 Entry

getBranch()
  = 从指定 leaf 回溯到 root 的路径

getTree()
  = 完整的树结构，供 TUI / RPC 展示

buildContextEntries()
  = 当前 leaf 对应的有效路径

buildSessionContext()
  = 把有效路径投影成 AgentMessage
~~~

当前模型 Context 只包含当前 leaf 到 root 的路径，不会包含同级的其他分支。

例如当前 leaf 是 assistant-B：

~~~text
模型看到：
root → user-1 → assistant-1 → user-branch-B → assistant-B

模型看不到：
user-branch-A → assistant-A
~~~

## 9. /tree：同一 Session 内的分支导航

交互式模式中的 /tree 会打开当前 Session 的树选择器：

~~~text
/tree
  ↓
查看当前 Session 的 Entry Tree
  ↓
选择历史节点
  ↓
可选：是否总结被离开的分支
  ↓
移动当前 leaf
  ↓
重新构建 agent.state.messages
  ↓
下一条输入从选中位置继续
~~~

### 选择不同类型节点的行为

如果选择 user Entry：

~~~text
目标 user Entry
  ↓
leaf 移动到该 user 的 parent
  ↓
该 user 的文本放回编辑器
  ↓
用户可以修改后重新发送
~~~

如果选择 assistant、toolResult 等非 user Entry：

~~~text
目标 Entry
  ↓
leaf 直接移动到目标节点
  ↓
下一条新的 user 输入从这里继续
~~~

因此，/tree 的作用是：

~~~text
同一个 Session 文件
  ↓
同一棵消息树
  ↓
在历史节点之间切换
  ↓
从任意节点继续生成新的子分支
~~~

原来的路径不会删除：

~~~text
root → A → B
          ├── C  原来的路径
          └── D  新的路径
~~~

在 D 上继续聊天后，还可以重新打开 /tree，选择 C 的末端回到原来的路径。

/tree 不会自动创建新的 Session 文件；它只是移动同一个 SessionManager 的 leaf。

主要代码：

~~~text
packages/coding-agent/src/modes/interactive/interactive-mode.ts:3039
packages/coding-agent/src/modes/interactive/interactive-mode.ts:5175
packages/coding-agent/src/core/agent-session.ts:3019
~~~

## 10. /tree、/fork、/clone、/resume 的区别

~~~text
/tree
  同一个 Session 文件内切换 leaf
  旧分支和新分支共存于一棵树

/fork
  从历史节点提取一条路径
  创建新的 Session 文件
  原 Session 仍然保留

/clone
  从当前 leaf 创建新的 Session 文件
  新 Session 从当前状态开始

/resume
  打开另一个已有 Session 文件
  创建对应的新 Agent Runtime
~~~

可以这样理解：

~~~text
/tree  = 在一棵树里切换分支
/fork   = 把某条路径复制成另一棵树
/resume = 切换到另一棵树
~~~

/tree 中的“回到最新位置”不是单独的自动恢复按钮，而是再次选择原来分支的最后一个节点。原分支仍然保留在树中。

## 11. Compaction 与 Session 的关系

Compaction 不会直接删除旧历史，而是追加一个 compaction Entry：

~~~text
原始历史：
A → B → C → D → E

追加：
A → B → C → D → E → Compaction(summary, firstKept=C)
~~~

重新构建 Context 时：

~~~text
Compaction Summary
        ↓
C → D → E
~~~

旧的 A、B 仍保留在 Session 文件中，但不再发送给模型。

因此：

~~~text
Session 文件
  = 更完整的历史和审计记录

Agent Context
  = 当前分支的压缩视图
~~~

Session Persistence 不等于 Context Management，但 compaction Entry 是两者之间的连接点。

## 12. Session 切换时的 Runtime 生命周期

切换 Session 时，Pi 不会把旧 Agent 直接换成另一组 messages，而是完整替换 Runtime：

~~~text
session_before_switch / session_before_fork
        ↓
abort 当前 Agent
        ↓
等待当前响应和工具结果 settle
        ↓
session_shutdown
        ↓
dispose 旧 AgentSession
        ↓
打开或创建新的 SessionManager
        ↓
创建新的 AgentSession Runtime
        ↓
重新绑定 UI / Extension / services
~~~

这样可以避免：

- 旧 Session 的事件继续写入新 Session；
- 旧 Extension Context 泄漏；
- 旧工具或旧 cwd 继续被使用；
- 正在执行的请求跨 Session 传播。

相关代码：

~~~text
packages/coding-agent/src/core/agent-session-runtime.ts:167
packages/coding-agent/src/core/agent-session-runtime.ts:196
~~~

## 13. 设计取舍

### 优点

- 历史记录不会因为分支或压缩被删除；
- Session 可以恢复、搜索、导出和展示；
- parentId 让对话天然支持分支；
- Agent Runtime 可以销毁后重新构建；
- 扩展可以保存自己的状态；
- Session 文件同时承担历史记录和 Context 重建输入的作用。

### 代价

- Session 文件会持续增长；
- 恢复时需要从树重新构建当前 Context；
- AgentState 与 SessionManager 之间必须保持同步；
- Session 文件保存的是消息和元数据，不是完整运行时；
- 正在流式生成的中间状态无法直接恢复；
- /tree、/fork、/resume 分别操作同文件分支、新文件和 Runtime 切换，概念较多。

## 14. 主要代码入口

~~~text
packages/coding-agent/src/core/session-manager.ts
packages/coding-agent/src/core/agent-session.ts
packages/coding-agent/src/core/agent-session-runtime.ts
packages/coding-agent/src/core/session-export.ts
packages/coding-agent/src/core/sdk.ts
packages/coding-agent/src/core/compaction/*
packages/coding-agent/src/modes/interactive/components/tree-selector.ts
~~~

