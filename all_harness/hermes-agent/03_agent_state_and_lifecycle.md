# Hermes Agent 03：状态、Turn 与生命周期

> 状态：已共同确认。本文是运行时骨架说明，不把它当作和 Memory、Tool、Compaction 同等级的独立机制。
>
> 研究日期：2026-08-31

## 1. 这一篇到底在讲什么？

这一篇只回答一件事：

> Hermes 怎样把一段长期会话切成许多次独立的用户运行，并让每一轮安全地开始、执行和结束。

它不是一个先有产品需求、再发明核心算法的模块。它是 Tool、Memory、Streaming、Retry、Nudge、Session 等能力逐步加进来后，为了让它们不互相污染而形成的运行时编排层。

最小 Agent 一开始只需要：

~~~text
history + user
  ↓
call model
  ↓
append assistant
  ↓
return
~~~

此时不需要 `TurnContext` 或 `TurnFinalizer`。

但当系统开始支持工具调用、持久化、流式输出、中断、Prompt Cache、Memory 注入、压缩和模型异常恢复时，每个功能都会要求一段明确的生命周期。于是自然会出现：

~~~text
AIAgent           长期持有会话运行状态
TurnContext        每条用户消息开始时准备本轮工作现场
Conversation Loop  在模型、工具和恢复分支之间推进
TurnFinalizer      在本轮结束时清理、修复和持久化
~~~

## 2. 四个运行单位

~~~text
AIAgent      = 长期存活的会话运行对象
SessionDB    = 跨进程、跨重启的持久化存档
Turn         = 从一条用户消息到最终交付的一整次运行
API call     = Turn 内的一次模型请求
~~~

例如“帮我修这个 bug”可以在一个 Turn 内经历多次模型调用和工具调用：

~~~text
用户消息
  ↓
模型思考 → read_file
  ↓
模型思考 → patch
  ↓
模型思考 → 最终回答
~~~

它们仍然是同一个用户 Turn，不是多个用户对话。

## 3. 几份消息状态不能混为一谈

| 状态 | 直白理解 | 生命周期 |
|---|---|---|
| `conversation_history` | 本轮开始前已有的历史基线 | Turn 输入 |
| `messages` | 本轮可变的工作稿，会加入 user / assistant / tool / 临时 nudge | Turn |
| `api_messages` | 某次实际发给模型的请求副本 | 单次 API call |
| `_session_messages` | 外壳保留的内存会话快照 | Session |
| `SessionDB` | 可恢复的持久化 transcript | Durable |

可以把它理解成：

~~~text
SessionDB        = 档案室
messages         = 桌面上的工作纸
api_messages     = 复印后递给模型的一份纸
~~~

临时 Memory、Plugin Context、Provider 字段和恢复材料可以进入 API 请求，但不应直接变成用户长期可见的聊天历史。

## 4. Nudge 为什么会自然需要这套结构？

假设模型只输出 reasoning、没有正文，Hermes 要用内部 nudge 继续推进：

~~~text
messages
  ├─ user：原始问题
  ├─ assistant：reasoning-only / 空回答 / 工具调用
  ├─ user：内部 nudge
  └─ assistant：最终正常回答
~~~

这立刻带来一组生命周期问题：

~~~text
nudge 算不算新用户消息？
  → 产品语义上不算；协议上通常要以 user role 发给模型

它要不要永久保存？
  → 不要，否则会污染真实 transcript

什么时候删？
  → 本轮真正结束前，由统一收尾点删除

删掉后 role sequence 还合法吗？
  → 必须检查；必要时闭合 assistant / tool 序列

中途崩溃怎么办？
  → 原始用户消息、真实 assistant/tool 仍要尽早持久化
~~~

因此，nudge 的正确位置是：

~~~text
同一个 run_conversation
  ├─ 可以发生多次 API call
  ├─ nudge 只存在于 working messages / API projection
  └─ finalizer 在持久化前清理内部 scaffolding
~~~

这也解释了“同一个逻辑 Turn”和“多次网络请求”为什么不矛盾。

## 5. Turn Prologue：开工前的职责

`build_turn_context()` 不创造业务能力，它负责让本轮以干净、可恢复的状态开始：

~~~text
恢复可能旋转后的 session
  ↓
重置本轮 retry / budget / stream 状态
  ↓
复制旧 history 并加入当前 user message
  ↓
恢复稳定的 system prompt
  ↓
必要时做预压缩、Memory Prefetch、Plugin Hook
  ↓
在第一次模型调用前持久化用户消息
~~~

重要区分：

- Turn 级状态每次新用户消息都会重置：retry 次数、iteration budget、当前 task/turn id、临时 context、stream callback。
- Session 级状态会保留：累计 token/cost、system prompt cache、memory/skill nudge 计数、SessionDB、Context Engine。

所以 `reset_session_state()` 和 `build_turn_context()` 不是一回事：前者是切换/重置会话，后者是每条用户消息开始时的准备。

## 6. Turn Finalizer：收工前的职责

`finalize_turn()` 是统一的收尾点。它不是主要负责“修复模型”，模型异常修复大多发生在 Loop 内；它负责把这一轮已经发生的事情变成一致、干净、可恢复的结果。

~~~text
Loop 退出
  ↓
清理 empty retry / nudge 等内部 scaffolding
  ↓
中断时修复 tool 调用的角色序列
  ↓
保证“已交付 final response ⇒ transcript 有 assistant 行”
  ↓
持久化 SessionDB，并更新内存 snapshot
  ↓
返回 final_response / reason / usage / failed / interrupted
~~~

它守住的不是花哨算法，而是几个关键不变量：

1. 用户已经看到的回答不能因为清理或持久化失败而消失。
2. 下一轮恢复的 transcript 必须角色合法，不能留下悬空 tool result。
3. 内部 retry/nudge 不应变成真正的聊天历史。
4. 已交付的回答必须能在 transcript 中找到，否则下一轮模型可能重复回答旧问题。

## 7. 真正值得学习的地方

这一层的价值不在于“应不应该有一个 Finalizer 类”，而在于它把分散功能的共同约束收束到了少数中心位置。

不好的演化方式通常是：

~~~text
empty retry 分支自己删消息
verification 分支自己补 assistant
interrupt 分支自己修 role
streaming 分支自己写 DB
tool 分支自己改 session
~~~

最后每个分支都直接改历史，功能之间会互相踩。

Hermes 的更成熟做法是：

~~~text
功能在 Loop 中提出自己的继续/恢复需求
  ↓
Working messages 承载临时状态
  ↓
Finalizer 作为统一出口清理和提交
~~~

因此，这一篇的正确定位是：

> 它是后续 Memory、Tool、Recovery、Session 等设计的支撑层；了解状态边界和收尾不变量即可，不必像 Memory 或特殊 Tool 一样深挖成独立产品机制。

## 8. 代码入口

- `run_agent.py`：`AIAgent` 的长期状态和 `run_conversation()` 公共入口。
- `agent/turn_context.py`：每个 Turn 的初始化、历史准备、预压缩、Memory/Plugin 注入和早期持久化。
- `agent/conversation_loop.py`：调用 prologue，运行主循环，最后进入 finalizer。
- `agent/turn_finalizer.py`：统一清理内部状态、修复 transcript、持久化并组装结果。