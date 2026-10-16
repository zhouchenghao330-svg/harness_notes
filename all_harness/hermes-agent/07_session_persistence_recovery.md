# Hermes Agent 07：Session / Persistence / Recovery

> 状态：已共同阅读并确认。  
> 本章重点：Hermes 如何把一次 Agent turn 变成可恢复的 durable transcript，并在进程崩溃、工具副作用、session 旋转和多进程并发下恢复到安全状态。

## 1. 核心结论

07 不是“把 messages 写进 SQLite”这么简单。

它真正维护的是一个 durable turn ledger：

~~~text
每一个可能影响后续模型调用、
工具副作用、会话恢复的状态
        ↓
都必须有明确的持久化边界
        ↓
进程重启后可以重新构造出一个安全的 turn
~~~

它需要回答：

~~~text
用户消息已经到达但模型还没调用，怎么办？
assistant 已经产生 tool_call 但工具结果还没回来，怎么办？
工具可能已经产生外部副作用，能不能重试？
两个 Hermes 进程同时操作同一个 session，怎么办？
context compression 之后 session_id 变了，如何找到真正的继续位置？
用户看到的内容和模型实际看到的内容不一致，如何恢复？
~~~

主要代码入口：

~~~text
hermes_state.py
  SessionDB、sessions/messages schema、lineage、locks、resume projection

gateway/session.py
  外部 session_key 到内部 session_id 的路由和恢复

run_agent.py
  Agent transcript flush、持久化 marker、可选 JSON snapshot

agent/turn_context.py
  turn 开始时创建 session、持久化 user message

agent/conversation_loop.py
  tool-call turn 在副作用前持久化

agent/tool_executor.py
  每个 tool result 完成后的增量持久化

agent/replay_cleanup.py
  中断工具和 dangling tool-call 的恢复清理
~~~

## 2. Hermes 里有几种不同的“会话身份”

不能把所有 ID 混成一个概念。

| 概念 | 含义 |
| --- | --- |
| session_key | 外部路由身份，例如 Telegram chat、Slack thread、用户会话 |
| session_id | 当前实际使用的物理 session segment |
| parent_session_id | session lineage，表示当前 session 从哪个 session 派生 |
| messages.id | 数据库中消息的永久插入顺序 |
| tool_call_id | assistant 工具调用和 tool result 的配对 ID |

整体关系：

~~~text
Telegram chat / Slack thread
          │
          ▼
     session_key
          │
          ▼
   当前 session_id
          │
          ▼
  SessionDB.messages
~~~

session_key 通常相对稳定，但 session_id 会发生变化：

~~~text
/new                 → 新 session_id
/branch              → 新 session_id + copied transcript
context compression  → continuation child session_id
session reset        → 新 session_id
delegate/subagent    → child session_id
~~~

因此 Hermes 把外部会话身份和内部 transcript segment 分离开来。

## 3. SessionDB 的数据模型

### 3.1 sessions 表：保存 session 身份和生命周期

sessions 行保存的不是消息正文，而是 session 级 metadata：

~~~text
session id / source / user / chat / thread
model / model_config
system_prompt / system_prompt_hash
parent_session_id
started_at / ended_at / end_reason
cwd / git branch / profile
token / cost / activity counters
compression / routing / handoff state
~~~

可以把它看成：

~~~text
Session row
├── 这个会话是谁
├── 它属于哪个外部聊天入口
├── 当前使用什么 model/provider 配置
├── 它从哪个 session 派生
├── 当前是否已经结束
├── 运行统计和工作目录是什么
└── 是否处于 compression / reset / handoff 等边界
~~~

system prompt 还会通过 hash 关联到 system_prompts 表，用于避免重复保存完整 prompt。

### 3.2 messages 表：保存可重放的 transcript

messages 行除了 role/content，还会保存：

~~~text
role
content
tool_call_id
tool_calls
tool_name
effect_disposition
finish_reason
reasoning
reasoning_content
reasoning_details
codex_reasoning_items
codex_message_items
api_content
display_kind
display_metadata
active / compacted / _compressed_summary
~~~

关键事实：

> Hermes 必须保存 assistant tool_calls、tool result、reasoning 和 Codex replay 信息，因为恢复 Agent 不能只依赖用户可见文本。

### 3.3 messages.id 比 timestamp 更适合作为恢复顺序

恢复 transcript 时使用 messages.id 的插入顺序，而不是 timestamp。

原因是 wall-clock 时间可能受到：

~~~text
NTP 校时
虚拟机时间跳变
电脑睡眠恢复
系统时钟不单调
~~~

影响，导致后写入的消息 timestamp 反而更早。

因此模型历史的顺序依据是：

~~~text
数据库插入顺序 messages.id
~~~

而不是 message.timestamp。

## 4. 一次正常 turn 的持久化流程

~~~text
外部消息到达
      │
      ▼
SessionStore:
session_key → session_id
      │
      ▼
turn_context
      ├─ 恢复或构造 system prompt
      ├─ 创建 sessions 行
      └─ append user message
             │
             ▼
       flush 到 SessionDB
             │
             ▼
        第一次 LLM call
             │
       ┌─────┴─────┐
       │           │
   普通文本       tool_calls
       │           │
       ▼           ▼
  final assistant   append assistant(tool_calls)
  最终持久化             │
                        ▼
              flush 到 SessionDB
                        │
                        ▼
                   执行工具
                        │
                        ▼
                 append tool result
                        │
                        ▼
              flush 到 SessionDB
                        │
                        ▼
                 下一次 LLM call
~~~

### 4.1 用户消息在第一次 LLM call 之前落库

turn_context 会在第一次 API call 之前持久化 inbound user turn。

因此：

~~~text
用户消息已经到达
模型请求还没有发出
Hermes 进程突然崩溃
~~~

恢复后仍然能知道用户刚才发送了什么。

### 4.2 assistant tool-call 在副作用之前落库

tool-call turn 的顺序是：

~~~text
append assistant(tool_calls)
        │
        ▼
flush SessionDB
        │
        ▼
执行工具副作用
~~~

而不是先执行工具、最后再保存 assistant(tool_calls)。

这个顺序专门防止：

~~~text
模型请求 terminal("docker restart")
        │
        ▼
Hermes 先执行 docker restart
        │
        ▼
Hermes 被重启，来不及保存 tool-call
        │
        ▼
恢复时完全不知道刚才执行过什么
~~~

### 4.3 tool result 增量持久化

工具结果不是等整个 tool batch 结束后才写。

通常是：

~~~text
tool A 完成
  ↓
append tool A result
  ↓
flush

tool B 完成
  ↓
append tool B result
  ↓
flush

tool C 完成
  ↓
append tool C result
  ↓
flush
~~~

这样即使后续工具失败或进程退出，也能保留已经完成的工具结果。

## 5. 持久化失败时为什么要 fail closed

Hermes 不允许 Agent 继续基于只存在于内存里的状态推进。

### assistant tool-call 持久化失败

~~~text
assistant(tool_calls)
        │
        ▼
写入 SessionDB 失败
        │
        ▼
不执行 side-effecting tools
停止当前 turn
~~~

### tool result 持久化失败

~~~text
工具已经执行
        │
        ▼
tool result 写入 SessionDB 失败
        │
        ▼
不把这个未落库的 result 发给模型
停止当前 turn
~~~

原因是：

~~~text
如果模型看到了一个不可恢复的 tool result，
下一次恢复时却看不到它，
模型上下文就会出现分叉。
~~~

这种设计会牺牲当前 turn 的完成度，但保护 transcript 一致性。

SessionDB 对 SQLite 写竞争会做应用层重试；如果最终仍然不能写入，则向上报告 session_persistence_failed，而不是把错误吞掉后继续执行。

## 6. 异常恢复流程

07 的恢复可以拆成五层：

~~~text
进程 / 数据库恢复
        ↓
session lineage 恢复
        ↓
tool-call / tool-result replay 清理
        ↓
role alternation / pairing 修复
        ↓
重新构造 model_history
        ↓
回到 Agent Loop
~~~

### 6.1 模型调用之前崩溃

~~~text
user message 已持久化
assistant response 尚未产生
~~~

恢复后 transcript 至少包含 user message，下一次 turn 可以继续处理这个请求。

### 6.2 assistant tool-call 已落库，但 tool result 不存在

例如：

~~~text
assistant(tool_calls: terminal restart)
~~~

Hermes 不能简单地让模型再次执行。

恢复规则：

~~~text
如果是只读工具
  → 删除未完成的 dangling assistant tool-call

如果可能产生外部副作用
  → 保留 assistant tool-call
  → 补一个 UNKNOWN tool result
  → 告诉模型：可能已经执行过，重试前先检查
~~~

### 6.3 工具已经返回，但结果显示 interrupted

例如：

~~~text
assistant(tool_calls)
tool: [command interrupted]
~~~

Hermes 会根据工具类型判断：

~~~text
read-only tool
  → 可以清理掉未完成 block

side-effecting tool
  → 不假设它没有执行
  → effect_disposition = unknown
  → 保留恢复提示
~~~

### 6.4 外部副作用不会自动回滚

Hermes 的恢复逻辑不是：

~~~text
工具失败
  → 回滚外部世界
  → 重新执行
~~~

而是：

~~~text
工具可能执行过
  → 标记 effect unknown
  → 先检查当前外部状态
  → 再由模型或用户决定是否重试
~~~

适用范围包括：

~~~text
文件写入
数据库修改
git 操作
发送消息
重启服务
执行 shell 命令
~~~

核心原则：

> interrupted 不等于 not executed。

### 6.5 replay cleanup 的整体路径

~~~text
SessionDB transcript
        │
        ▼
gateway/replay loader
        ├─ 清理 gateway 注入的 auto-continue noise
        ├─ 清理 interrupted tool tail
        ├─ 处理 dangling assistant(tool_calls)
        └─ 清理过期的危险确认文本
        │
        ▼
get_resume_conversations()
        ├─ model_history
        │    └─ repair_alternation=True
        └─ display_history
        │
        ▼
pre-call sanitizer
        ├─ 修复非法 role
        ├─ 清理 orphan tool result
        ├─ 补齐缺失的 tool result stub
        ├─ 去重 tool_call_id
        └─ 修复空内容消息
        │
        ▼
发给 Provider
~~~

这里的 repair 不一定修改数据库原始 transcript。

很多修复只发生在给模型的 replay projection，或者本次 API call 的临时 copy 中，避免为了修复一次请求而破坏原始审计记录。

## 7. model_history 和 display_history

恢复时，Hermes 会从同一次 lineage 查询构造两套 projection：

~~~text
同一批 durable message rows
             │
       ┌─────┴─────┐
       │           │
model_history   display_history
       │           │
给模型继续跑    给 UI 展示
~~~

### model_history

~~~text
当前 tip 的 active rows
修复 role alternation
修复 tool-call / tool-result 配对
保留 api_content
保留 reasoning / Codex replay fields
适合继续进入 Agent Loop
~~~

### display_history

~~~text
可以包含 compression ancestors
保留完整 timeline
保留 display_kind
保留 display_metadata
适合 CLI / Gateway / TUI / Desktop 展示
~~~

因此：

~~~text
model_history ≠ display_history
~~~

这和常见的：

~~~text
checkpointer 保存完整状态
messages 表保存用户可见内容
~~~

是同一个思想。

但 Hermes 没有完全拆成两张物理消息表，而是通过一条 message row 的多个字段表达不同投影：

~~~text
content
  = 主要持久化内容

api_content
  = 实际发给模型的精确内容

display_kind
  = 是否隐藏、如何渲染

display_metadata
  = UI 展示元数据
~~~

## 8. api_content：模型侧和用户侧内容分离

当前轮 user message 可能被临时拼接：

~~~text
memory recall
plugin context
prefetch context
system-side note
~~~

这些内容可能需要进入模型请求，但不一定希望污染用户看到的 transcript。

因此 Hermes 会保存：

~~~text
content
  = 干净的持久化内容

api_content
  = 实际发送给 Provider 的完整内容
~~~

恢复时，如果 api_content 存在，就用它重放模型侧内容，避免：

~~~text
本轮发送给模型的内容
        ≠
恢复后重新发送给模型的内容
~~~

否则从这一条开始，后续 prompt cache 前缀都会发生变化。

这也是 04 Context / Prompt / Prompt Cache 和 07 Persistence 的交界：

~~~text
04 决定模型实际看到什么
07 确保模型侧 projection 可以被恢复
~~~

## 9. Session lineage：compression、branch、reset 不是一回事

~~~text
root session S0
      │
      ├── context compression
      │       └── S1
      │           parent = S0
      │           S0.end_reason = compression
      │
      ├── /branch
      │       └── B1
      │           parent = S0
      │           _branched_from = S0
      │           copied transcript
      │
      ├── /new / reset
      │       └── S2
      │           _reset_from = S0
      │           fresh transcript
      │
      └── delegate / subagent
              └── D1
                  parent = S0
~~~

它们都可能使用 parent_session_id，所以还需要：

~~~text
_branched_from
_reset_from
_delegate_from
end_reason
~~~

来区分 lineage 语义。

### 9.1 compression child

compression child 通常承接：

~~~text
summary
+ 最近仍然需要的尾部
+ compression 之后的新消息
~~~

旧 parent 会以 end_reason = compression 结束。

恢复时不能只看用户选中的旧 session_id，而要找到 compression continuation tip。

### 9.2 branch child

branch 是用户主动创建的独立分支：

~~~text
旧 session
   │
   ├─ 保留原 transcript
   └─ 复制一份 transcript 到新 session
~~~

之后两个 session 独立发展。

复制时 Hermes 会尽量保留：

~~~text
tool_calls
tool_call_id
reasoning fields
Codex fields
api_content
timestamp
~~~

这样 branch 的第一轮也可以继续复用原来的模型侧内容。

### 9.3 new / reset

/new 或 reset 不是继续 parent，而是建立新的 conversation boundary：

~~~text
旧 session
  → end_reason = new_session / session_reset

新 session
  → 没有旧 transcript
  → 新的 session_id
  → session-scoped model/runtime state 重新初始化
~~~

## 10. /resume 的完整流程

/resume 不是简单读取一个文件。

~~~text
用户执行 /resume
        │
        ▼
解析 session id / title / recent index
        │
        ▼
检查目标 session 是否是 compression 旧 head
        │
        ├─ 是 → 找到 continuation tip
        └─ 否 → 使用目标 session
        │
        ▼
flush 当前 session 中还没写入的消息
        │
        ▼
结束当前 session
        │
        ▼
读取 model_history + display_history
        │
        ▼
切换当前 session_id
        │
        ▼
reopen 目标 session
        │
        ▼
恢复 AIAgent runtime
        ├─ session_id
        ├─ conversation_history
        ├─ cwd
        ├─ model/provider
        ├─ YOLO/approval state
        └─ session-local runtime state
        │
        ▼
下一次用户输入从恢复后的 transcript 继续
~~~

model_history 用于后续 API call，display_history 用于展示完整恢复时间线。

## 11. Gateway 的 session_key → session_id 路由

Gateway 不能每次收到消息都重新猜 session。

它维护：

~~~text
外部 session_key
        │
        ▼
SessionStore routing entry
        │
        ▼
当前 session_id
        │
        ▼
SessionDB transcript
~~~

同时它还处理：

~~~text
idle reset
daily reset
restart recovery
compression tip healing
stale routing entry
session key migration
legacy sessions.json 兼容
并发请求 single-flight
~~~

同一个 session_key 的并发创建会共享同一个 owner 的结果：

~~~text
请求 A ─┐
请求 B ─┼─→ single-flight
请求 C ─┘       │
                ▼
        只创建一个 session
~~~

如果 Gateway 重启，它会把最近活跃、可能被中断的 session 标记为：

~~~text
resume_pending = true
resume_reason = restart_interrupted
~~~

下一次同一 session_key 的消息到达后，Gateway 会尝试继续原 transcript，而不是默认创建全新会话。

## 12. durable turn lease：防止多个进程同时跑一个 session

Hermes 支持：

~~~text
CLI
Gateway
TUI
Desktop
后台 delivery
subagent / worker
~~~

它们可能共享同一个 state.db。

因此 Hermes 使用 session_turn_leases，把下面这一整段串行化：

~~~text
load history
    ↓
run model/tool loop
    ↓
flush transcript
~~~

并发时：

~~~text
进程 A:
获得 turn lease
  ↓
加载 history
  ↓
执行 turn
  ↓
flush
  ↓
释放 lease

进程 B:
发现 lease 被占用
  ↓
等待
  ↓
A 释放
  ↓
重新加载最新 history
  ↓
执行自己的 turn
~~~

lease 绑定的是 conversation lineage root，而不是简单的当前 session segment。

这样 context compression 在中途把 S0 旋转到 S1 时，仍然不会让两个进程同时操作同一个用户会话。

如果 lease 丢失或无法刷新，Hermes 会停止当前 turn：

~~~text
Session turn lease lost
        ↓
stop to protect the transcript
~~~

而不是继续写入可能已经过时的 transcript。

## 13. SQLite、WAL 和 FTS5

SessionDB 是 SQLite-backed session storage。

设计目标是：

~~~text
多个 reader
一个 writer
WAL 提高并发读取能力
应用层重试缓解 write contention
~~~

它还维护 FTS5：

~~~text
messages_fts
  ├─ content
  ├─ tool_name
  └─ tool_calls
~~~

另外还有 trigram index，用于 CJK 等语言的 substring search。

因此 SessionDB 同时承担：

~~~text
durable transcript
session search
历史召回
token/cost statistics
routing metadata
~~~

## 14. state.db、JSON snapshot 和 gateway routing mirror

需要区分三种存储：

~~~text
state.db
  = canonical session metadata + messages + routing + locks

session_{session_id}.json
  = 可选的 per-session JSON snapshot
    供外部工具消费，不是 canonical store

sessions.json
  = Gateway routing 的 legacy mirror
    主要用于兼容旧版本
~~~

run_agent.py 中的 JSON snapshot 由 sessions.write_json_snapshots 控制，默认不是主路径。

因此不能把 JSON snapshot 当成 Hermes 真正的 session source of truth。

## 15. 和其他组件的边界

### 02 Agent Loop

02 负责：

~~~text
本次 turn 如何调用模型
何时调用工具
何时继续下一轮
何时结束
~~~

07 负责：

~~~text
把这些关键状态写下来
崩溃后重新构造
~~~

### 02.5 Agent Loop Repair

02.5 处理：

~~~text
thinking 截断
empty response
没有正文
异常 tool-call
continuation / nudge / retry
~~~

07 处理：

~~~text
进程崩溃
数据库未写完
tool side effect 未确认
session 旋转
session 恢复
多进程并发
~~~

### 03 Agent State / Lifecycle

03 讨论 Agent 在内存中的：

~~~text
process state
session state
turn state
finalization
~~~

07 讨论这些状态如何进入 durable storage，并在下次进程中重新绑定。

### 04 Context / Prompt / Prompt Cache

04 决定：

~~~text
模型实际看到什么
哪些内容属于 stable prefix
哪些内容属于当前 turn
~~~

07 保证模型侧的历史 projection 可以被精确恢复。

### 08 Memory / Context Engine / Compaction

08 负责：

~~~text
memory
token budget
context compression
summary
session rotation
~~~

07 负责保存并恢复：

~~~text
compression lineage
parent/child session
summary carrier
active/inactive rows
resume tip
~~~

所以 07 不负责设计压缩算法，但负责让压缩后的 session 仍然可恢复。

## 16. 最值得记忆的设计原则

### 原则一：持久化边界放在副作用之前

~~~text
先保存“我要做什么”
再执行外部动作
~~~

### 原则二：外部副作用不能默认可回滚

~~~text
interrupted ≠ not executed
~~~

所以要使用 effect_disposition = unknown，而不是盲目 retry。

### 原则三：外部身份和内部 segment 分离

~~~text
session_key
  = 用户从哪个外部会话来

session_id
  = 当前 transcript 存在哪个物理 segment
~~~

### 原则四：模型历史和展示历史分离

~~~text
model_history
  = 为了正确 replay

display_history
  = 为了正确展示
~~~

### 原则五：恢复是 replay reconstruction，不是一次 retry

~~~text
恢复数据库状态
        ↓
找到正确 session tip
        ↓
清理 dangling tool
        ↓
修复消息协议
        ↓
恢复 model projection
        ↓
重新进入 Agent Loop
~~~

### 原则六：持久化失败时宁可停止，也不要继续产生不可恢复状态

~~~text
写不进去
  → 不继续执行未落库的副作用
  → 不继续向模型投喂未落库结果
  → fail closed
~~~

## 17. 和项目设计的对应关系

如果项目中已经有：

~~~text
LangGraph checkpointer
messages 表
~~~

可以这样映射：

~~~text
LangGraph checkpointer
  ≈ Hermes sessions + messages + lineage + locks

用户可见 messages
  ≈ Hermes display_history / content / display_kind

模型精确 replay
  ≈ Hermes api_content + tool_calls + reasoning fields

session 恢复路由
  ≈ Hermes session_key → session_id
~~~

Hermes 的特点不是物理上拆成很多表，而是在 canonical SessionDB 中同时保留：

~~~text
完整模型轨迹
用户展示 projection
session lineage
工具副作用状态
恢复所需的并发信息
~~~

这使得 07 成为 Agent 的 durable boundary，而不是简单的聊天记录模块。

## 18. 代码阅读地图

~~~text
hermes_state_common.py
  sessions/messages/system_prompts/FTS schema

hermes_state.py
  SessionDB
  create/end/reopen session
  append message
  batch flush
  get_resume_conversations
  compression tip
  lineage
  turn lease

gateway/session.py
  session_key routing
  single-flight create
  reset policy
  restart resume
  gateway recovery

run_agent.py
  _ensure_db_session
  _persist_session
  _flush_messages_to_session_db
  _DB_PERSISTED_MARKER
  optional JSON snapshot

agent/turn_context.py
  turn-start DB creation
  inbound user persist
  api_content staging

agent/conversation_loop.py
  assistant tool-call pre-side-effect persist
  persistence failure stop

agent/tool_executor.py
  incremental tool-result persist

agent/replay_cleanup.py
  interrupted/dangling tool recovery
~~~

## 19. 一句话总结

07 的本质是：

~~~text
让 Agent 的每个关键动作都有一个 durable、可审计、
可恢复、不会盲目重复副作用的边界。
~~~

它不是单纯的“保存历史”，而是：

~~~text
Session Identity
+ Durable Transcript
+ Replay Reconstruction
+ Side-effect Recovery
+ Lineage
+ Concurrency Ownership
~~~

共同组成 Hermes 的会话恢复系统。
