# Hermes Agent 02：Agent Loop

> 状态：已按当前 checkout 的代码完成第一版阅读。本文只讲“正常路径”：一次用户输入如何被推进成模型响应、工具执行、下一次模型调用，最后收束成最终回答。异常停止、thinking-only、截断、nudge、replay、fallback 和 stop gate 统一放在 [[02.5_agent_loop_repair]]。
>
> 研究日期：2026-08-28

## 1. 它解决什么问题？

Hermes 的 Agent Loop 不是“一次请求进来、一次请求出去”的薄封装。它要把一个用户请求推进成一段可能包含多次模型调用的运行：

~~~text
用户输入
  ↓
模型调用
  ↓
模型给出工具调用
  ↓
执行工具
  ↓
把工具结果写回消息历史
  ↓
再次调用模型
  ↓
模型给出最终文本
  ↓
持久化并结束本次运行
~~~

因此，Loop 的核心问题是：

- 当前模型响应是工具调用，还是可以交给用户的最终文本？
- 如果是工具调用，哪些工具现在执行，哪些必须等前面的工具结束？
- 工具结果如何成为下一次模型调用的输入？
- 当前这次用户运行什么时候才真正结束？
- 在不污染历史和不破坏 Prompt Cache 的情况下，如何构造下一次 API 请求？

它不等于完整的 Hermes Harness，也不负责单独设计：

- 所有 System Prompt 内容；
- 工具本身的业务逻辑；
- Provider 的完整 HTTP 实现；
- SessionDB 的全部存储细节；
- 异常恢复策略的完整集合。

这些能力由 Loop 调用的上下文、Transport、Tool Executor、Session 和 Recovery 组件提供。

## 2. 先把几个“循环单位”分开

Hermes 代码里容易把几个概念混在一起。按当前实现，可以这样理解：

~~~text
一次 run_conversation（一次用户级运行 / outer turn）
  ├─ API call slot 1
  │    ├─ provider request
  │    └─ 同一个请求槽位里的 transport retry（如果发生）
  ├─ API call slot 2
  │    ├─ provider request
  │    └─ tool result 之后的下一轮调用
  ├─ API call slot 3
  └─ finalization
~~~

- `run_conversation()` 是外部看到的一次完整用户运行。
- 一个用户运行内部可以有很多次 Provider API call。
- `api_call_count` 主要记录外层循环已经消耗了多少个调用槽位；同一个槽位里的网络/API retry 由 `retry_count` 管理。
- 一个“工具轮次”通常是一个 assistant tool-call 消息，加上它产生的全部 tool result。
- 只有通过最终化阶段后，才是这次用户运行的真正结束。

这个区别很重要：调用模型一次，不等于整个 Agent 运行结束；模型返回一条文本，也不一定已经通过所有停止条件。后一个问题在 [[02.5_agent_loop_repair]] 详细展开。

## 3. 正常路径总图

~~~text
CLI / Gateway / TUI / Desktop / ACP / API / Cron
                  │
                  │ user_message + session context
                  ▼
        AIAgent.run_conversation()
                  │
                  ▼
        build_turn_context()
        ├─ 恢复 Session / 压缩后的历史
        ├─ 加入当前 user message
        ├─ 取得稳定的 system prompt
        ├─ 初始化本轮 budget / callback / state
        └─ 运行 turn-start 逻辑
                  │
                  ▼
        构造本次 API request copy
        ├─ clean messages
        ├─ system prompt
        ├─ active tools
        └─ API-only context / provider projection
                  │
                  ▼
        Provider Transport / API Client
                  │
                  ▼
        normalize_response()
                  │
        ┌─────────┴─────────┐
        │                   │
   有结构化 tool_calls       没有 tool_calls
        │                   │
        ▼                   ▼
   校验 / 规范化          得到最终候选文本
        │                   │
        ▼                   ├─ 正常停止检查
   先持久化 assistant       ├─ append final assistant
   tool-call 消息            ├─ flush SessionDB
        │                   └─ 离开 Loop
        ▼
   规划工具 batch
        │
        ▼
   执行工具
   ├─ safe segment：可并行
   └─ barrier segment：按顺序
        │
        ▼
   append + persist tool results
        │
        └──────────────→ 下一次模型调用
~~~

上图是“模型响应已经被正常解析”的主链。连接错误、响应被截断、只输出 reasoning、工具调用丢失等情况会在这条主链的若干检查点被转交给修复分支，而不是直接当成成功。

## 4. Turn Prologue：循环真正开始前做什么？

Loop 不是拿到 `user_message` 就直接请求模型。`run_conversation()` 先调用 `build_turn_context()`，把长期 Agent 状态投影成这一次运行需要的局部变量：

~~~text
现有 Agent / Session 状态
          ↓
    build_turn_context()
          ├─ session 恢复 / 旋转
          ├─ conversation_history → messages 副本
          ├─ 追加当前 user message
          ├─ 恢复或构造 cached system prompt
          ├─ 预取 memory / plugin context
          ├─ 预压缩（必要时）
          ├─ 建立本轮 persistence / turn state
          └─ 返回 Loop 使用的局部状态
~~~

这里要特别记住两条约束：

1. **System Prompt 的稳定性优先。** Hermes 把长期 system prompt 缓存起来，正常的每次模型调用都复用它。临时上下文优先放到 API-call-time 的 user message 或 request projection 中，避免每轮重建 system prompt 破坏 Prompt Cache。
2. **Canonical history 和 API request copy 分开。** `messages` 是运行时的规范历史；`api_messages` 是本次发送前复制出来的请求视图。Provider 字段清洗、cache marker、`api_content` sidecar、reasoning echo 和临时上下文都应该在请求视图上处理。

## 5. 一次正常 API 调用的准备过程

每次进入外层循环，Hermes 大致按下面顺序准备请求：

~~~text
messages（规范历史）
  ↓
tool-call argument sanitize
  ↓
清理历史中的幽灵行 / 修复角色序列
  ↓
结构化 clone
  ↓
移除 display-only / persistence-only 字段
  ↓
注入本轮 API-only context
  ↓
根据 Provider 处理 reasoning_content / reasoning_details
  ↓
构造 system + messages + tools
  ↓
清理孤儿 tool result、规范 tool-call JSON、清理非法字符
  ↓
根据 Provider 生成 cache layout
  ↓
build_api_kwargs()
  ↓
调用 Transport
~~~

这不是在修改“真实聊天记录”。例如：

- `api_content` 是为了记住历史消息当初真正发给 API 的内容，不是 Provider 字段；
- `_row_id`、`display_kind`、`finish_reason` 等内部字段不会直接发给严格 API；
- Memory / plugin 的临时召回可以进入当前 user message 的 API 副本，但不会把临时文本写回干净的 transcript；
- cache marker 最后才加，确保前面的清洗完成后，连续调用之间的稳定前缀保持一致。

源码在 `agent/conversation_loop.py:2164-2535` 附近，尤其要把 `messages` 和 `api_messages` 当成两个不同层次看。

## 6. Provider 返回后：先归一化，再决定下一步

不同 API mode 的原始响应形状不同：

~~~text
chat_completions
anthropic_messages
bedrock_converse
codex_responses
          │
          ▼
   Transport.normalize_response()
          │
          ▼
统一的 assistant_message
  ├─ content
  ├─ reasoning / reasoning_content
  ├─ tool_calls
  ├─ finish_reason
  └─ provider-specific replay state
~~~

Loop 后面的分支主要看两个事实：

1. `assistant_message.tool_calls` 是否真的有结构化调用对象；
2. 如果没有，是否存在可以交给用户的可见文本。

模型在 reasoning 里写了“我现在要调用工具”，不等于第 1 条成立。只有归一化结果里的结构化 `tool_calls` 才会进入 Tool Executor。这个设计避免把普通文本误当成可执行动作；它的异常补救和边界情况见 [[02.5_agent_loop_repair#11.2 dropped tool call guard]]。

## 7. 正常 Tool Call 路径

### 7.1 先校验调用，再触发副作用

正常的工具调用链是：

~~~text
assistant_message.tool_calls
          │
          ├─ 工具名是否在 valid_tool_names？
          ├─ 参数是否是合法 JSON？
          ├─ call id 是否唯一、可配对？
          ├─ 是否需要去重 / 限额？
          └─ 是否经过 guardrail / approval？
          │
          ▼
构造 assistant(tool_calls) 消息
          │
          ▼
先写入 SessionDB
          │
          ▼
再执行工具副作用
~~~

“先持久化、后执行”是恢复能力的基础：如果终端、浏览器或其他副作用工具在执行中让进程退出，恢复时至少能看到模型已经发出的完整 tool-call block，而不是一个只存在于内存中的动作。

### 7.2 一个 batch 如何执行？

Hermes 不把“多工具调用”简单地全部并行或全部串行。`_execute_tool_calls()` 先把同一条 assistant 消息里的调用切成连续 segment：

~~~text
模型发出的原始顺序：

[read_file] [search] [write_file] [clarify] [read_file]
     safe       safe       barrier      barrier      safe

执行计划：

parallel segment 1: [read_file, search]
          ↓ 全部结束
sequential barrier: [write_file]
          ↓ 完成
sequential barrier: [clarify]
          ↓ 完成
parallel segment 2: [read_file]
~~~

规则是：

- 单个调用直接走顺序路径；
- 连续的 read-only、非重叠文件目标或明确 opt-in 的工具可以组成并行 segment；
- interactive、unsafe、未识别或有资源冲突的调用是顺序 barrier；
- segment 按模型原始顺序执行，前一个 barrier 完成前，后面的 segment 不启动；
- 并行工具的完成事件可以先后不同，但写回模型的结果必须能和原始 `tool_call_id` 正确配对；
- 中断时，未执行的调用也会获得取消/跳过结果，避免留下没有对应 tool result 的 assistant tool-call。

unsafe 在这里不是“代码里看到 `unsafe=True` 就一定如此”的单一字段，而是调度器判断“不能和同批其他调用无条件并行”的类别。工具安全分类本身属于 Tool System；本篇只记录 Loop 如何消费这个分类。

### 7.3 Tool Result 写回并回到模型

~~~text
执行工具
  ↓
得到 result / error / cancelled result
  ↓
append role=tool + tool_call_id
  ↓
每个结果尽快 flush SessionDB
  ↓
把 steer / budget 等控制信息附着到合法结果位置
  ↓
activity touch
  ↓
下一次模型调用
~~~

下一次请求不是“重新从头问模型”，而是把：

~~~text
... 历史
assistant(tool_calls=[...])
tool(result for call A)
tool(result for call B)
~~~

作为新的 API request copy 发回 Provider，让模型根据观察结果决定下一步。只要模型继续返回结构化 tool calls，Loop 就重复这一段。

## 8. 正常最终文本路径

当归一化响应没有结构化 `tool_calls` 时，它会进入 no-tool 分支，形成最终候选：

~~~text
assistant response
  ↓
没有 tool_calls
  ↓
提取可见文本
  ↓
移除内部 reasoning / think 标记
  ↓
通过停止前检查
  ↓
构造 assistant final message
  ↓
flush SessionDB
  ↓
离开外层 Loop
~~~

这里的“通过停止前检查”故意没有展开成“只要有文本就结束”。Hermes 在最终化前还可能检查：

- 模型是否只说了“我接下来会做某事”却没有行动；
- Provider 是否声称有 tool call 但实际数组为空；
- 编辑后的工作区是否有新鲜验证证据；
- Kanban worker 是否发出了规定的终止工具。

这些检查不是普通工具轮次的一部分，但它们决定文本能不能真的成为最终答案，详见 [[02.5_agent_loop_repair#11. 为什么没有 tool call 仍然不一定结束？——Stop Gates]]。

## 9. 外层 Loop 和最终化

代码中的主骨架可以压缩成：

~~~python
while budget_allows:
    build_api_request_copy()
    response = provider_call()
    assistant_message = normalize_response(response)

    if assistant_message.tool_calls:
        append_assistant_tool_call()
        persist_before_side_effects()
        execute_tool_calls()
        persist_tool_results()
        continue

    final_candidate = visible_text(assistant_message)
    if final_candidate_passes_stop_checks(final_candidate):
        append_final_assistant(final_candidate)
        persist()
        break

return finalize_turn(...)
~~~

这段伪代码省略了异常分支，但保留了 Hermes Loop 的控制关系：**模型决定下一步是工具还是文本，工具结果成为下一次模型输入，最终化统一收束所有出口。**

`finalize_turn()` 负责把剩余的预算收束、trajectory、资源清理、Session 持久化、结果字典和后续 review 触发集中处理。正常文本路径在 `agent/conversation_loop.py:8402-8424` 先做一次 final message flush，再进入 finalizer；这样进程若在 finalizer 前被关闭，已经展示给用户的答案也尽量不会丢。

## 10. Agent Loop 必须守住的几个不变量

### 10.1 工具调用和结果必须成对

~~~text
assistant(tool_calls=[A, B])
  ↓
tool(A)
tool(B)
~~~

不能只留下 `tool(A)`，也不能让下一条 user message 直接接在孤立 `tool` 后面。否则严格 Provider 可能返回 400，宽松 Provider 也可能静默返回空内容。

### 10.2 发送视图可以变，规范历史不能被无意污染

发送前的清洗、临时上下文、Provider 字段和 cache marker 应该作用于 `api_messages`。Hermes 的许多恢复 bug，本质都是把“为某个 Provider 临时修请求”错误地写回了 canonical history。

### 10.3 System Prompt 和工具面尽量稳定

长会话依赖稳定的 Prompt Cache。正常循环不会因为每次 tool result 就重建整个 system prompt，也不会因为一个 API 端点的 wire 形状变化而随意更换全套工具。

### 10.4 工具执行副作用和模型继续调用分开

先记录 assistant tool-call，再执行副作用；工具结果写回并持久化后，才允许下一次模型调用。这样“模型想做什么”“工具实际上返回什么”“会话已经保存什么”三个事实不会混成一个临时字符串。

### 10.5 完成事件晚于真实完成

流式 delta、reasoning callback、tool progress 是实时投影；SessionDB 和 `finalize_turn()` 是持久化/生命周期收束。UI 看见 token，不代表 canonical transcript 已经写完；UI 看见 tool progress，也不代表工具 batch 已经全部结束。

## 11. 代码证据

| 代码位置 | 作用 |
| --- | --- |
| `run_agent.py:8586-8605` | `run_conversation()` 把外层结果交给 `finalize_turn()` |
| `agent/conversation_loop.py:1823-2018` | Loop 入口、turn 状态和外层 budget loop |
| `agent/conversation_loop.py:2164-2535` | 从 canonical messages 构造 API request copy |
| `agent/conversation_loop.py:6760-7042` | Provider response normalize、incomplete 检查和响应分派 |
| `agent/conversation_loop.py:7042-7460` | tool-call 校验、assistant tool-call 持久化和工具执行入口 |
| `run_agent.py:8417-8459` | 单工具、并行 batch、混合 segment 的调度选择 |
| `agent/tool_executor.py:1932-1950` | 取消/中断时补齐 tool result |
| `agent/tool_executor.py:2864-2921` | mixed batch 按 parallel/sequential segment 执行 |
| `agent/conversation_loop.py:7678-7685` | no-tool 响应进入最终候选分支 |
| `agent/conversation_loop.py:8402-8424` | final assistant flush 和正常 Loop 退出 |
| `agent/turn_finalizer.py:121-390` | 预算、清理、持久化和结果收束 |

## 12. 本篇先不展开的问题

- Provider 请求失败后，到底是重试、换 credential、压缩还是 fallback？
- thinking-only 和 incomplete reasoning 如何区分？
- 为什么有时要把已经生成的 reasoning 作为 interim 重新放回请求？
- 为什么 nudge 明明写着 `[System: ...]`，却使用 `role=user`？
- 为什么没有 tool call 仍然可能继续模型调用？
- UI 已经显示 reasoning 后，重试时是否会重复显示？

这些都属于恢复层，而不是正常的模型→工具→结果主链，统一见 [[02.5_agent_loop_repair]]。