# Hermes Agent Harness 拆解总览

> 状态：第一轮粗扫，待共同阅读。以下是当前 checkout 的 Agent-centric 地图，不是最终组件结论。
>
> 研究日期：2026-08-28

## 1. 研究定位

Hermes Agent 是一个被多个外壳复用的个人 AI Agent：

- 交互式 CLI
- Messaging Gateway
- TUI / Dashboard
- Electron Desktop
- ACP 编辑器集成
- API Server、Cron、Batch 和 Delegation

本次研究不做 Hermes 全产品架构导览，而是拆出它如何把模型变成一个可执行、可持续、可恢复的 Agent Runtime。

重点关注：

- 用户输入如何进入一次 Agent turn；
- 模型请求如何构造、发送和归一化；
- Agent 如何在模型、工具和观察结果之间循环；
- Context、System Prompt 和 Prompt Cache 如何保持稳定；
- Tool 如何注册、选择、校验、执行和回写；
- State、History、Memory、Compaction 和 Session 如何协作；
- Stop、Retry、Cancel、Approval、Budget 和 Delegation 如何改变控制流；
- 事件、流式输出和最终回复如何离开 Runtime 到达不同外壳。

## 2. 研究边界

### 2.1 第一轮纳入

只要代码会改变以下任一项，就属于 Hermes Harness：

- 模型输入或输出；
- System Prompt、工具描述或运行时上下文；
- Agent Loop、turn 和调用次数；
- Tool Call、Tool Result、权限和执行顺序；
- Session、History、Memory、Context Engine；
- Compaction、Summary、Checkpoint、Recovery、Replay；
- Stop、Retry、Cancel、Approval、Timeout、Iteration Budget；
- Provider、API Mode、请求格式和响应归一化；
- Event、Streaming、Callback、SSE、JSON-RPC 输出；
- Plugin、Skill、MCP、Subagent 对 Agent 行为的注入或拦截。

### 2.2 第一轮暂时排除

暂时不深入：

- 不改变 Agent 行为的前端布局和纯 UI 样式；
- 各消息平台的普通收发、登录和 API 细节；
- 安装、发布、计费后台和不影响 Runtime 的管理 CRUD；
- 通用部署和基础设施实现；
- 具体 Provider 的全部 HTTP 细节。

如果这些代码改变了模型看到什么、工具能做什么、会话如何恢复、运行是否继续，仍然要把它影响 Agent 的部分纳入。

## 3. 当前主运行路径

Hermes 的入口很多，但它们最终汇入同一个有状态的 AIAgent 外壳：

~~~text
CLI / Gateway / TUI / Desktop / ACP / API / Cron / Batch
  │
  ├─ CLI：HermesCLI.chat()
  ├─ Gateway：TurnRunner.run_sync()
  ├─ TUI：prompt.submit
  ├─ ACP：ACP session handler
  └─ 其他外壳：创建或复用 AIAgent
  │
  ↓
AIAgent
  │  run_agent.py
  │  agent.agent_init.init_agent()
  │
  ├─ model / provider / api_mode
  ├─ tool snapshot / valid_tool_names
  ├─ cached system prompt
  ├─ ContextCompressor / Memory / Context Engine
  ├─ SessionDB binding
  ├─ interrupt / steer / redirect / budget
  └─ stream / tool / status / event callbacks
  │
  ↓
AIAgent.run_conversation()
  │  run_agent.py 只是公共入口和状态外壳
  │
  └─ agent.conversation_loop.run_conversation()
       │
       ├─ build_turn_context()
       │    恢复 Session、准备 user message、加载/恢复 Prompt、
       │    运行 pre-turn 逻辑、准备 Memory / Plugin 上下文
       │
       ├─ 默认 Python Agent Loop
       │    构造 API 请求副本
       │    → 调用 Provider
       │    → 归一化响应
       │    → Tool Call 或最终文本
       │
       └─ api_mode == codex_app_server
            → 转交 Codex app-server runtime
~~~ 

## 4. Agent-centric 架构图

~~~text
┌──────────────────────────────────────────────────────┐
│                    外部运行外壳                      │
│ CLI / Gateway / TUI / Desktop / ACP / API / Cron    │
└──────────────────────┬───────────────────────────────┘
                       │ 用户输入、会话标识、回调
                       ↓
┌──────────────────────────────────────────────────────┐
│              Session / Surface Binding               │
│ session_key ↔ session_id                            │
│ platform / profile / enabled_toolsets               │
│ SessionDB / gateway session state / runtime cache    │
└──────────────────────┬───────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│                      AIAgent                         │
│                                                      │
│ model / provider / api_mode                          │
│ tools / valid_tool_names                             │
│ _session_messages / cached_system_prompt             │
│ context_compressor / memory / callbacks              │
│ interrupt / steer / redirect / iteration budget      │
└──────────────────────┬───────────────────────────────┘
                       │ run_conversation(user_message)
                       ↓
┌──────────────────────────────────────────────────────┐
│                 Turn Prologue                        │
│ agent/turn_context.py                                │
│                                                      │
│ 恢复压缩后的 Session                                  │
│ 复制历史并追加当前 user message                       │
│ 恢复或构造缓存 System Prompt                          │
│ 预取 Memory / Plugin context                          │
│ 检查 MCP / Session / pre-turn 状态                    │
└──────────────────────┬───────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│              Conversation Loop                       │
│ agent/conversation_loop.py                           │
│                                                      │
│  interrupt / redirect / budget                       │
│       ↓                                              │
│  clean history → API request copy                    │
│       ↓                                              │
│  system prompt + messages + tools                    │
│       ↓                                              │
│  Provider transport / client                         │
│       ↓                                              │
│  normalize response                                  │
│       ├───────────────┬──────────────────────────────┤
│       │               │                              │
│   最终文本        Tool Call                        │
│       │               │                              │
│       │       参数修复 / 校验 / 去重 / Guardrail      │
│       │               ↓                              │
│       │        Tool Executor / Tool Router            │
│       │               ↓                              │
│       │        Tool Result 写回 messages              │
│       │               └──────── 回到 Loop              │
│       ↓                                              │
│  Turn Finalizer / result                             │
└──────────────┬───────────────────┬───────────────────┘
               │                   │
               ↓                   ↓
┌────────────────────────┐  ┌─────────────────────────┐
│ Context / Memory /     │  │ Control / Recovery      │
│ Compaction             │  │ Retry / Cancel / Approval│
│                         │  │ Budget / Fallback       │
│ ContextCompressor       │  │ Delegate / Subagent      │
│ conversation_compression│  │                         │
│ ContextEngine / Memory │  │                         │
└─────────────┬───────────┘  └────────────┬────────────┘
              │                           │
              └──────────────┬────────────┘
                             ↓
┌──────────────────────────────────────────────────────┐
│ Persistence / Events / Output                         │
│ SessionDB / JSON snapshot / trajectory / logs         │
│ stream_delta / tool callbacks / event_callback        │
│ CLI display / Gateway delivery / TUI JSON-RPC / ACP   │
└──────────────────────────────────────────────────────┘
~~~

## 5. 一次用户输入的初步调用链

~~~text
用户输入
  ↓
外壳绑定 session_key / session_id 和 platform
  ↓
创建或复用 AIAgent
  ↓
AIAgent.run_conversation()
  ↓
agent.turn_context.build_turn_context()
  ├─ 恢复压缩轮转后的 Session
  ├─ 复制 conversation_history
  ├─ 追加当前 user message
  ├─ 恢复或构建 cached system prompt
  ├─ 预取外部 Memory / Plugin 上下文
  └─ 做 turn-start 持久化和状态初始化
  ↓
agent.conversation_loop.run_conversation()
  ↓
while iteration_budget / max_iterations 允许
  ├─ 检查 interrupt / redirect / steer / run budget
  ├─ 从 clean messages 构造 API request copy
  │    ├─ cached system prompt
  │    ├─ conversation messages
  │    ├─ active tool definitions
  │    └─ API-only context injection
  ├─ transport.build_kwargs()
  ├─ interruptible API call / streaming call
  ├─ transport.normalize_response()
  ├─ 无 Tool Call → final response / finalize
  └─ 有 Tool Call
       ├─ 校验工具名和 JSON 参数
       ├─ Tool Guardrail / Approval
       ├─ 顺序或并发执行
       ├─ append tool result
       └─ 回到下一次模型调用
  ↓
持久化、统计、事件和外壳输出
~~~

这里的 “一次运行” 需要在后续共同阅读中继续精确定义：当前代码对外暴露的是一次 run_conversation 调用；其内部可以包含多次 Provider API call、Tool Call 和重试/压缩分支。不要直接把 API call、turn、run 三个概念混为一谈。

## 6. Hermes 的两个运行分支

### 6.1 默认 Python Loop

默认路径由 Hermes 自己持有完整控制权：

~~~text
AIAgent
  ↓
build_turn_context
  ↓
agent.conversation_loop
  ↓
Provider transport / client
  ↓
Tool execution
  ↓
Session / Memory / Compression / Output
~~~

这个分支需要重点分析 Hermes 自己如何维护消息、工具、压缩和恢复。

### 6.2 Codex app-server 分支

当 api_mode 为 codex_app_server 时，turn 会绕过默认的 Python Tool Loop，进入 agent.codex_runtime 与 Codex app-server session。

这不是普通 Provider 格式转换，而是另一种运行时边界：

~~~text
Hermes 外壳
  ↓
AIAgent / turn setup
  ↓
Codex app-server runtime
  ↓
Codex 自己持有部分 Thread / Tool / Compaction 生命周期
~~~

这一分支后续单独研究，不能和默认 Python Loop 混成一条证据链。

## 7. 第一轮判断

### 7.1 AIAgent 是状态外壳，不是全部循环

run_agent.py 的 AIAgent 负责：

- 对外提供 chat() 和 run_conversation()；
- 保存模型、Provider、API Mode、工具、Session 和回调状态；
- 暴露 interrupt、steer、redirect、session persistence 等控制入口；
- 把初始化和长方法委托给 agent/ 下的专门模块。

真正的默认对话循环已经抽到：

- agent/turn_context.py；
- agent/conversation_loop.py；
- agent/turn_finalizer.py；
- agent/tool_executor.py；
- agent/chat_completion_helpers.py。

因此后续不能只读 run_agent.py，也不能只看 agent/ 目录名就默认所有模块同等重要。

### 7.2 Hermes 的缓存边界是显式设计

当前代码明确把两类数据分开：

~~~text
Session / clean messages
  ↓ 结构化复制
API request copy
  ├─ provider-specific sanitization
  ├─ api_content sidecar replay
  ├─ memory / plugin API-only injection
  └─ ephemeral system prompt
~~~

初步观察：

- cached system prompt 按 AIAgent 生命周期复用；
- System Prompt 通常只在首次构建、Session restore 或 Compaction 后重建；
- API request 使用独立副本，避免 provider 清洗和 cache decoration 回写持久历史；
- Memory / Plugin 的临时上下文优先进入 API-facing user message，而不是随意改写 System Prompt；
- 这条边界直接影响 Prompt Cache、Session 恢复和重试，必须作为后续核心专题。

### 7.3 工具系统有三个不同层次

~~~text
工具模块 / Plugin / MCP 注册
  ↓
Tool Registry：工具存在什么、属于哪个 toolset、是否可用
  ↓
Toolset Resolver + check_fn：本 Session 暴露什么
  ↓
get_tool_definitions()：发给模型的 schema
  ↓
handle_function_call() / agent runtime：宿主如何执行
  ↓
Tool Result：如何回到下一次模型请求
~~~

“已注册”不等于“当前 Session 可见”，
“模型可见”也不等于“工具执行成功”。后续要分别读这三个边界。

### 7.4 SessionDB 不只是数据库封装

SessionDB 同时承载：

- Session 元数据和 model config；
- 消息追加、历史读取和 FTS 搜索；
- token / cost / activity 统计；
- compression lineage、child session 和 tip；
- compression lock、turn lease 和并发保护；
- Session 恢复、轮转、失败冷却和清理。

因此 Persistence 不是 Agent Loop 之后的普通存储层，而是控制运行连续性的 Harness 组件。

### 7.5 Hermes 同时存在多个状态载体

初步需要区分：

- AIAgent 内存状态：当前工具、模型、回调、压缩器、interrupt 和消息快照；
- clean transcript：准备持久化和恢复的消息；
- API request copy：发送给 Provider 的副本，可能含 API-only sidecar；
- SessionDB：跨进程、跨重启的 durable 状态；
- Gateway / TUI session state：外壳的 session_key、队列和输出状态；
- Memory / Context Engine 状态：压缩与外部记忆的附加状态。

后续要确认这些状态何时同步、谁是权威、失败时如何回滚。

## 8. 第一轮暂时排除的产品细节

- CLI 的纯渲染和输入快捷键；
- Gateway 各平台的 API 适配细节；
- Dashboard / Desktop 的页面布局；
- 具体 MCP Server 的业务功能；
- Provider 的所有认��和 HTTP 实现；
- 不改变 Agent 输入、输出、控制流和恢复语义的运维代码。

但以下情况要重新纳入：

- 外壳改变 session_key 或 conversation_history；
- 平台决定工具集、权限、Approval 或是否允许继续；
- TUI / Gateway 改变事件和流式输出语义；
- Provider 适配改变消息、工具、缓存或 reasoning 的重放；
- 插件、Skill、MCP 改变 Prompt 或 Tool surface。

## 9. 第一阶段建议研究顺序

先共同阅读：

1. Agent Loop：agent/conversation_loop.py
2. Agent State / Turn Context：run_agent.py、agent/agent_init.py、agent/turn_context.py
3. Context / Prompt / Cache：agent/system_prompt.py、agent/prompt_builder.py、agent/prompt_cache_scope.py
4. Tool System：model_tools.py、tools/registry.py、toolsets.py、agent/tool_executor.py
5. Session / Persistence：hermes_state.py、gateway/session.py、run_agent.py

之后再看：

6. Compaction / Context Engine / Memory
7. Model / Provider / Transport
8. Control / Retry / Cancel / Approval / Budget
9. Delegation / Subagent / Background Review
10. Events / Streaming / Output
11. Extensions / Skills / Plugins / MCP
12. Surface Capability / Gateway / TUI session binding

## 10. 当前代码证据入口

- pyproject.toml:378-379：hermes CLI entry point
- cli.py:5001：HermesCLI
- cli.py:16270：CLI chat
- gateway/run.py:4293：TurnRunner
- gateway/run.py:5365：TurnRunner.run_sync
- gateway/run.py:6537：Gateway 调用 AIAgent.run_conversation
- tui_gateway/methods_prompt.py:287：prompt.submit
- run_agent.py:421：AIAgent
- run_agent.py:8586：AIAgent.run_conversation 公共转发入口
- agent/agent_init.py:init_agent：Agent 运行时状态与 Provider/Tool/Compression 初始化
- agent/turn_context.py:496：turn prologue
- agent/conversation_loop.py:1823：默认完整对话循环
- agent/system_prompt.py:341、910：System Prompt 分层组装和缓存入口
- model_tools.py:323：get_tool_definitions
- model_tools.py:1240：handle_function_call
- tools/registry.py:111：内置 Tool discovery
- tools/registry.py:763：Tool registry.register
- agent/transports/base.py:16：Provider transport 抽象
- agent/chat_completion_helpers.py：API 调用、stream、retry 和 mode dispatch
- hermes_state.py:4385：SessionDB
- hermes_state.py:11061：append_message
- hermes_state.py:11950：get_messages
- hermes_state.py:12298：get_messages_as_conversation

后续每个组件笔记都必须回到这些源码入口确认，不把本文件的初步地图当作最终证据。
