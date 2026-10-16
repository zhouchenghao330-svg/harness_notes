# Hermes Agent Harness 组件索引

> 研究边界：只按 Agent 行为拆解，不按仓库目录做“全项目导览”。
>
> 当前状态：第一轮代码粗扫，组件地图已建立；各组件仍需和你一起逐个读代码确认。

## 1. 组件组织原则

Hermes 的组件是否属于本次拆解，取决于它是否改变以下 Agent 行为：

- 模型输入输出：请求 payload、响应归一化、流式增量、provider/API mode。
- 上下文：system prompt、历史消息、当前轮输入、缓存前缀、压缩后的恢复上下文。
- 工具：工具发现、工具集、schema、可用性检查、调用、结果、审批和错误处理。
- 状态：AIAgent 状态、turn 状态、SessionDB、会话恢复、并发锁和重试。
- 记忆与上下文管理：memory、context engine、压缩、token/budget。
- 控制流：迭代上限、中断、取消、deadline、重试、redirect/steer、优雅收尾。
- 扩展：skills、plugins、MCP、subagent/delegation。
- 产出：事件、stream、gateway/TUI callback、最终回答和持久化。

每个组件后续都用同一组问题阅读：

1. 它的唯一职责是什么？
2. 谁创建它、谁持有它、谁调用它？
3. 它输入什么，输出什么？
4. 它改变哪些 state？
5. 它是否改变模型可见的 prompt/tools/API payload？
6. 它和 SessionDB 的关系是什么？
7. 它的生命周期是 process、session、turn 还是单次 API call？
8. 它有哪些失败、恢复、并发或缓存约束？

## 2. 组件总表

| 编号  | 组件                                           | 主要代码入口                                                                                                                                                                                                             | Agent-centric 作用                                     | 状态             |
| --- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- | -------------- |
| 01  | Entry / Session / Run Orchestrator           | cli.py、gateway/run.py、tui_gateway/methods_prompt.py、run_agent.py                                                                                                                                                   | 把 CLI、消息平台、TUI 等外壳绑定到 session，并创建或复用 AIAgent         | 待共同阅读          |
| 02  | Agent Loop / Conversation Loop               | agent/conversation_loop.py                                                                                                                                                                                         | 驱动一次 turn 内的 API call、tool call、tool result 和继续条件    | 待共同阅读          |
| 03  | Agent State / Turn Context / Finalization    | run_agent.py、agent/agent_init.py、agent/turn_context.py、agent/turn_finalizer.py                                                                                                                                     | 区分长期 Agent 状态、单轮状态、轮前准备和轮后收尾                         | 待共同阅读          |
| 04  | Context / Prompt / Prompt Cache              | agent/system_prompt.py、agent/prompt_builder.py、agent/prompt_cache_scope.py、agent/turn_context.py                                                                                                                   | 组装模型上下文并维护长会话的稳定缓存前缀                                 | 待共同阅读          |
| 05  | Tool Registry / Toolset / Tool Execution     | model_tools.py、tools/registry.py、toolsets.py、agent/tool_executor.py                                                                                                                                                | 发现、筛选、暴露、校验、审批、执行工具并回填结果                             | 部分完成（框架已读，核心工具待共同阅读） |
| 06  | Model / Provider / Transport                 | agent/transports/、agent/chat_completion_helpers.py、provider adapters                                                                                                                                               | 把内部消息和工具转换成不同 API 格式，并归一化响应                          | 待共同阅读          |
| 07  | Session / Persistence / Recovery             | hermes_state.py、gateway/session.py、run_agent.py                                                                                                                                                                    | 持久化 transcript、索引、统计、压缩 lineage、恢复和并发租约              | 待共同阅读          |
| 08  | Memory / Context Engine / Compaction         | tools/memory_tool.py、agent/memory_provider.py、agent/memory_manager.py、agent/background_review.py、agent/context_engine.py、agent/context_compressor.py、agent/conversation_compression.py、agent/native_compaction.py  | 区分长期记忆、请求级上下文选择和有损压缩，并维护 cache、checkpoint、持久化与恢复边界   | 已共同阅读          |
| 09  | Control / Retry / Cancel / Approval / Budget | agent/iteration_budget.py、agent/turn_retry_state.py、agent/retry_utils.py、agent/error_classifier.py、tools/interrupt.py、tools/approval.py、agent/deadline.py                                                          | 将 logical iteration、物理 retry、取消传播、人工授权、执行期限和耗尽兜底分层管理 | 已共同阅读          |
| 10  | Delegation / Subagent / Background Run       | tools/delegate_tool.py、tools/async_delegation.py、tools/subagent_worktree.py、tools/delegation_output_schema.py、agent/subagent_lifecycle.py、agent/delegation_context.py、agent/background_review.py、agent/moa_loop.py | 生成子 Agent、隔离权限与上下文、并行/后台执行、结果回流，以及 MoA 多顾问推理         | 已共同阅读          |
| 11  | Events / Streaming / Output                  | agent/stream_single_writer.py、agent/stream_diag.py、gateway/stream_consumer.py、gateway/stream_events.py、tui_gateway/event_publisher.py                                                                              | 把内部 API/tool/turn 状态安全地投影到 CLI、Gateway、TUI 等消费者      | 跳过（输出投影层，按需回看） |
| 12  | Extensions / Skills / Plugins / MCP          | hermes_cli/plugins/、plugins/、agent/skill_commands.py、agent/skill_preprocessing.py、tools/mcp_tool.py、gateway/hooks.py                                                                                               | 在不膨胀核心工具面的前提下扩展能力、提示和外部服务                            | 已共同阅读          |
| 13  | Surface Capability / Platform Binding        | toolsets.py、model_tools.py、gateway/session_context.py、gateway/session.py、tui_gateway/server.py                                                                                                                     | 根据 session 的 platform/source 决定 GUI、消息平台等能力是否可见      | 已共同阅读          |

## 3. Hermes 的初步分层

~~~text
外壳层
  CLI / Gateway / TUI / Dashboard / Electron / ACP / API / Cron / Batch
        │
会话与路由层
  session_key ↔ session_id
  platform / profile / enabled_toolsets / session context
        │
Agent 外壳
  AIAgent
  model / provider / api_mode / callbacks
  cached system prompt / messages / compressor / memory / controls
        │
每轮上下文层
  turn_context
  恢复或旋转 session
  清理输入、重置 turn state、预算、memory nudge
  取历史、构建或复用 system prompt、建立 DB turn
        │
执行循环层
  API request copy
    ├─ provider transport / API mode / retry
    ├─ response normalize
    ├─ final text → finalize
    └─ tool calls
         ├─ validate / guardrail / approval
         ├─ execute
         ├─ append tool result
         └─ continue loop
        │
横切 runtime
  compaction / memory / recovery / deadline / cancellation
  delegation / subagent / plugins / skills / MCP
        │
持久化与投影层
  SessionDB / transcript / stats / lineage
  callbacks / stream events / gateway / TUI output
~~~

## 4. 第一阶段阅读顺序

先把决定“Agent 怎么跑”的核心链读通，再展开横切能力：

1. agent/conversation_loop.py：一次 turn 如何循环。
2. agent/turn_context.py + agent/turn_finalizer.py：一次 turn 如何开始和结束。
3. agent/system_prompt.py + prompt cache：哪些内容稳定，哪些内容进入当前轮。
4. model_tools.py + tools/registry.py + toolsets.py：模型看到的工具从哪里来。
5. hermes_state.py + gateway/session.py：历史和 session 如何保存、恢复、旋转。
6. 再读 compaction/memory、provider/transport、control/retry、delegation、events、extensions。

这里的顺序不是按 import 顺序，而是按“模型一次调用为什么会产生下一步行为”来排。

## 5. 当前已确认的主代码边界

### run_agent.py

AIAgent 的公共外壳、运行时 state、控制入口、持久化/回调兼容，以及对内部模块的转发。它很重要，但不能把它等同于完整 Agent Loop。

### agent/conversation_loop.py

默认的 Python Agent Loop：检查 budget/interrupt，构造本次 API 请求，调用 provider，归一化响应，处理文本或 tool calls，并决定是否继续。

### agent/turn_context.py

turn prologue：处理 session 恢复/旋转、MCP 刷新、输入清理、turn 状态和预算、历史消息、memory nudge、system prompt 缓存、DB turn 和必要的预压缩。

### agent/turn_finalizer.py

turn finalization：把最终响应、usage、状态、错误、统计、持久化和回调收束到一致的结果上。细节待共同阅读。

### agent/agent_init.py

初始化 model/provider/api_mode、工具定义、回调、interrupt/steer/redirect、prompt cache、SessionDB、memory/context compressor 和各类 client。

### model_tools.py / tools/registry.py / toolsets.py

分别覆盖工具定义编排、运行时注册/发现、toolset 与可用性筛选，以及函数调用分发。后续需要严格区分“注册了什么”“本 session 暴露什么”“实际执行了什么”。

### agent/transports/ + agent/chat_completion_helpers.py

transport 负责 provider 之间的消息、工具、请求参数和响应转换；helper 负责 client、API mode、stream/retry/fallback 等调用路径。需要继续确认两者的职责边界。

### hermes_state.py / gateway/session.py

SessionDB 不只是“保存聊天记录”：还涉及 FTS 搜索、token/cost/activity、压缩 lineage、子 session、锁/租约和恢复；gateway/session 负责外部 session key 与内部 session_id 的映射和 transcript 生命周期。

### gateway/run.py / tui_gateway/ / cli.py / acp_adapter/

它们是不同入口和输出面，真正进入 Agent 行为的部分要沿着 session、AIAgent、turn 和事件链追踪；UI 布局本身不纳入核心拆解。

## 6. 本轮拆解纪律

- 先读真实代码，再把确认后的结论写入 Obsidian。
- 每条结论标记为：代码确认、文档描述或当前推断。
- 不把 CLI、Gateway、TUI、Desktop 当成 Agent 核心本身。
- 不把 run_agent.py 当成全部循环，也不把 agent/ 下所有文件默认视为同等核心。
- 分开讨论 history、memory、context、compaction，不能用“上下文”一词混成一件事。
- 对工具追踪完整生命周期：discovery → schema → active toolset → model call → validation → execution → result → persistence/event。
- 阅读 provider 时看最终 API payload，而不是只看 adapter 类名。
- 单独记录 codex_app_server：它是可选的另一条运行分支，不要和默认 provider 格式转换混为一谈。
- 当前只建立索引，不预填后续所有组件的结论。

## 7. 后续文档结构

后续按组件拆成：

- 02_agent_loop.md
- 03_agent_state_and_lifecycle.md
- 04_context_and_prompt.md
- 05_tool_system.md
- 06_session_and_persistence.md
- 07_compaction_and_memory.md
- 08_model_and_provider_adapter.md
- 09_control_and_cancellation.md
- 10_delegation_and_subagents.md
- 11_events_streaming_and_output.md
- 12_extensions_skills_plugins_mcp.md
- 13_surface_capability_platform_binding.md

下一步先在聊天里共同拆 02_agent_loop.md，确认后再落详细笔记。
