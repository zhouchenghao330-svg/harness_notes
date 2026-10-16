# Hermes Agent 12：Extensions / Skills / Plugins / Middleware / MCP

> 状态：已共同阅读并确认。  
> 本章重点：Hermes 如何让外部能力在不持续修改 Agent Loop 的前提下进入 Runtime；尤其区分 Plugin、Registry、Hook、Policy、Transformer、Middleware、Event Bus 与客户端事件。  
> 阅读视角：Skills 已较熟悉、MCP 暂不展开；重点放在 Plugin / Middleware，以及这套设计对 AI-Human Runtime 的启发。  
> 代码复核：基于 2026-09-15 当前 checkout。正文会明确区分“当前源码事实”“Hermes 的历史包袱”和“面向 AI-Human 的建议”。

## 1. 为什么这一章对 AI-Human 很重要

当 Runtime 还在早期时，大部分需求都可以直接写进 Agent Loop：

~~~text
Agent Loop
    ↓
构造上下文
    ↓
调用模型
    ↓
解析 tool call
    ↓
执行工具
    ↓
输出事件
~~~

但 Runtime 一旦逐渐稳定，真正的问题就会发生变化：

~~~text
A 同事：我要增加一个业务工具
B 同事：我要给每轮模型请求插入动态业务上下文
C 同事：我要在工具调用前补 tenant_id
D 同事：我要修改 temperature、路由模型或增加 fallback
E 同事：我要向前端发送一种新的业务事件
~~~

如果没有正式的扩展体系，这些需求最后都会变成：

~~~python
if app == "equipment":
    ...

if business_type == "writing":
    ...

if tool_name == "query_company_data":
    ...
~~~

然后业务逻辑不断渗入：

~~~text
agent_loop.py
workflow.py
tool_executor.py
stream.py
router.py
~~~

因此这一章的核心问题不是“怎样加载一个 Plugin”，而是：

> Runtime 已经逐渐稳定，但业务能力还会持续变化。怎样允许其他开发者扩展 Runtime，而不需要进入 Agent Loop 修改核心代码？

Hermes 最值得学习的地方，是它已经开始把两件事拆开：

~~~text
稳定的 Core Runtime
        │
        │ 提供明确扩展边界
        ▼
持续变化的业务能力
~~~

## 2. 核心结论：Plugin 不是一种执行机制

最容易混淆的是把所有扩展能力都叫作 Plugin 或 Hook。

实际上：

> Plugin 只是一个业务能力的安装包和统一注册入口；真正改变 Runtime 的，是它注册进去的 Registry、Hook、Middleware、Provider、Prompt Section、Skill 等对象。

先记住这张映射表：

| 扩展诉求 | 更准确的机制 | 典型语义 |
| --- | --- | --- |
| 增加一种能力 | Registry | 注册 tool、provider、command、skill、prompt section |
| 某件事发生时通知我 | Observer Hook | trace、metrics、audit、logging |
| 在某节点作有限决策 | Policy / Directive | allow、deny、modify、require approval、continue |
| 修改即将使用的数据 | Transformer / Request Middleware | 改请求参数、工具参数、结果或文本 |
| 包住真正的执行过程 | Execution Middleware | fallback、cache、retry、circuit breaker、trace |
| Plugin 之间通信 | Event Bus | 进程内发布订阅 |
| 观察模型 token 流 | Stream Observer | 异步消费 delta，不阻塞主输出 |
| 给客户端发送业务事件 | Runtime Event Sink | schema、seq、持久化、回放、SSE/WS 投影 |

Hermes 当前源码没有把这些概念完全隔离干净，但已经能看到清晰的演进方向：

~~~text
增加能力       → Registry
观察事实       → Observer Hook
改变请求       → Request Middleware
接管执行       → Execution Middleware
内部通信       → Event Bus
观察 token     → Stream Observer
客户端事件     → 仍缺少通用 Plugin API
~~~

## 3. Plugin 的真实角色：业务能力安装包

Hermes 加载 Plugin 后，会给它一个 `PluginContext`。Plugin 通过这个受控入口注册能力，而不是直接拿到整个 `AIAgent` 随意修改。

~~~text
Plugin
  │
  │ register(ctx)
  ▼
PluginContext
  ├── register_tool()
  ├── register_cli_command() / register_command()
  ├── register_system_prompt_section()
  ├── register_hook()
  ├── register_middleware()
  ├── register_skill()
  ├── register_context_engine()
  ├── register_memory_provider()
  ├── register_image_gen_provider()
  ├── register_web_search_provider()
  ├── register_browser_provider()
  ├── register_terminal_environment_provider()
  ├── register_tts_provider() / register_transcription_provider()
  ├── register_platform()
  ├── emit() / subscribe()
  └── spawn_task()
~~~

例如一个 AI-Human 业务 Plugin 可以表达为：

~~~python
def register(ctx):
    ctx.register_tool(query_equipment)
    ctx.register_system_prompt_section(equipment_prompt)
    ctx.register_hook("post_tool_call", record_equipment_trace)
    ctx.register_middleware("tool_request", inject_tenant_id)
~~~

Runtime 不需要理解 Equipment 是什么。它只认识：

~~~text
Tool Registry
Prompt Registry
Hook Registry
Middleware Registry
Provider Registry
~~~

这就是 Plugin 的价值：

> 它把一个业务模块需要注册的多种扩展组合在一起，但不把所有扩展语义压扁成一个万能 `hook()`。

## 4. Registry：新增能力，不要修改 Loop

Registry 解决的是：

> Runtime 现在拥有哪些可被选择的能力？

例如 Plugin 注册工具后，基本关系是：

~~~text
Plugin
  ↓ register_tool()
Process Tool Registry
  ↓ 当前 session / app / platform 的能力解析
Agent Tool Snapshot
  ↓ 转换成 tools schema
Provider Request
~~~

这里必须区分：

~~~text
Process Registry
  这个进程里有哪些候选能力

Session / Agent Snapshot
  当前这个 Agent 实际暴露哪些能力
~~~

这对 AI-Human 很关键。以后可以是：

~~~text
全局 Capability Registry
  ├── query_equipment
  ├── query_case
  ├── writing_tools
  └── search_kb
             │
             ▼
       Capability Resolver
      app / tenant / session
             │
             ▼
equipment app 的 Agent Snapshot
  ├── query_equipment
  └── search_kb
~~~

而不是每个业务复制一套 Agent Loop。

需要注意 Hermes 的两条长期约束：

1. 工具 schema 会进入模型请求，不能因为进程中注册了工具，就让所有 session 都无条件承担 token 成本。
2. 同一会话的 system prompt 和工具面应尽量稳定；会话中途随意增删工具会破坏前缀缓存和行为一致性。

所以 Registry 只回答“有什么”，Capability Resolver 才回答“本 session 能看到什么”。

## 5. Hook：这里发生了一件事

最纯粹的 Hook 是 Observer：Runtime 到达某个生命周期点时，把已经发生的事实通知订阅者。

~~~text
Tool 执行完成
      ↓
post_tool_call
      ├── Plugin A：记录 trace
      ├── Plugin B：统计 metrics
      └── Plugin C：写审计日志
~~~

理想的 Observer 契约是：

~~~python
callback(event) -> None
~~~

它适合：

- tracing；
- logging；
- metrics；
- billing；
- audit；
- debug；
- lifecycle bookkeeping。

Hermes 的 Hook payload 会携带 session、turn、tool call、API request、task 等关联标识，使 Plugin 可以拼出：

~~~text
Run
 └── Turn
      ├── LLM Request 1
      ├── Tool Call 1
      ├── LLM Request 2
      └── Final Answer
~~~

这比让外部 Plugin 解析日志文本可靠得多。

### 5.1 Hermes 的 Hook 已经不只是 Observer

当前 `VALID_HOOKS` 中同时存在几种不同语义：

~~~text
Observer
  post_tool_call
  post_llm_call
  on_stream_*
  approval / skill / subagent / kanban 生命周期

Transformer
  transform_terminal_output
  transform_tool_result
  transform_llm_output
  transform_api_error_classification

Policy / Directive
  pre_tool_call
  pre_verify
  pre_gateway_dispatch

Dynamic Context
  pre_llm_call

Provider Boundary Observer / Extension
  pre_api_request
  post_api_request
  api_request_error
~~~

也就是说，Hermes 的 `Hook` 是一个历史形成的总称：有的只观察，有的返回替换值，有的能 block，有的能要求 Agent 继续。

这套实现能工作，但对于新 Runtime 不值得照抄。因为一旦都叫 Hook，就会出现：

~~~text
返回值有没有意义？
多个 Plugin 的返回值怎么合并？
谁先执行？
一个 allow、一个 deny 时谁赢？
超时是继续还是阻止？
异常是 fail-open 还是 fail-closed？
~~~

AI-Human 更适合从一开始就把类型分开：

~~~text
Observer
  callback(event) -> None

Transformer
  transform(value, ctx) -> value

Policy
  evaluate(input, ctx)
    -> Allow | Deny | Modify | RequireApproval | Continue

Middleware
  invoke(request, next_call) -> response
~~~

## 6. Middleware：进入真实数据流

Middleware 解决的不是“通知”，而是：

> 我要改变 Runtime 即将执行的内容，或者包住真正的执行过程。

Hermes 当前定义了四类 middleware：

| 类型 | 作用点 | 能做什么 |
| --- | --- | --- |
| `llm_request` | provider 调用前 | 修改最终模型请求 payload |
| `llm_execution` | provider 执行边界 | 包装、替换或短路真实模型调用 |
| `tool_request` | policy / guardrail 前 | 修改最终工具参数 |
| `tool_execution` | tool handler 执行边界 | 包装、替换或短路真实工具执行 |

这四类可进一步归成两组：

~~~text
Request Middleware
  改“即将执行什么”

Execution Middleware
  改“怎样执行它”
~~~

### 6.1 Request Middleware：修改有效请求

假设模型产生：

~~~json
{
  "tool": "query_company_data",
  "args": {"id": 123}
}
~~~

业务 Plugin 可以补充运行时事实：

~~~python
def inject_tenant(args, ctx):
    return {
        "args": {
            **args,
            "tenant_id": ctx.tenant_id,
        }
    }
~~~

最终执行的是：

~~~json
{
  "id": 123,
  "tenant_id": "tenant_xxx"
}
~~~

这里最重要的不是“能改参数”，而是它在安全检查之前运行：

~~~text
模型原始 tool args
        ↓
tool_request middleware
        ↓
最终有效 args
        ↓
pre_tool_call policy
        ↓
guardrail / approval
        ↓
tool execution
~~~

审批必须检查最终会执行的参数。不能先批准 path A，再让 Plugin 把它改成 path B。

同理，`llm_request` 适合修改：

- provider kwargs；
- temperature / max tokens；
- routing hint；
- provider-specific extra body；
- 当前 API request 的附加字段。

### 6.2 Execution Middleware：包住真正执行

Execution Middleware 使用洋葱模型：

~~~text
Tracing Middleware
  └── Fallback Middleware
       └── Cache Middleware
            └── Real Provider
~~~

执行顺序是：

~~~text
Tracing.before
  Fallback.before
    Cache.before
      Provider()
    Cache.after
  Fallback.after
Tracing.after
~~~

契约大致是：

~~~python
def middleware(request, next_call):
    before()
    response = next_call(request)
    after(response)
    return response
~~~

它可以实现：

- model routing；
- fallback；
- cache；
- retry；
- circuit breaker；
- mock；
- tracing；
- 限流；
- 统一错误映射。

它也可以直接短路：

~~~python
def cache_middleware(request, next_call):
    cached = cache.get(request)
    if cached is not None:
        return cached
    return next_call(request)
~~~

Hermes 对每一层的 `next_call()` 做了 single-use 限制。一次 middleware frame 如果调用两次，会抛出契约错误，避免无意中执行两次 Provider 或工具副作用。

### 6.3 Middleware 的异常语义

当前 execution chain 的行为是：

~~~text
middleware 在调用 next_call 前报错
  → 记录 warning
  → 跳过这一层，继续 downstream

middleware 调用 next_call，downstream 成功，随后自己报错
  → 保留 downstream 已产生的结果
  → 不能重新执行 downstream

downstream 本身报错
  → 原始错误继续向上传播

同一层调用 next_call 两次
  → RuntimeError，禁止二次执行
~~~

这个处理保护了“真实执行至多一次”的关键不变量。

但当前 middleware 没有像 Agent 热路径 Hook 那样配置独立 timeout。第三方 middleware 一旦挂死，仍可能阻塞模型或工具执行。这是 AI-Human 设计时必须补上的边界。

### 6.4 当前 Request Middleware 不是严格的逐层 Reducer

这里有一个值得注意的实现细节。

理想的顺序组合通常是：

~~~text
request0
  ↓ Plugin A
request1
  ↓ Plugin B
request2
~~~

但 Hermes 当前 `_invoke_middleware()` 会先用同一份本轮 payload 调用已注册 callback，收集返回值，再按注册顺序选取各 callback 返回的完整 replacement。

因此它更接近：

~~~text
Plugin A sees request0 → returns requestA
Plugin B sees request0 → returns requestB

reduce results in registration order
  request0 → requestA → requestB
~~~

而不是让 Plugin B 自然看到 Plugin A 已经修改后的 requestA。若多个 Plugin 同时替换完整 payload，后注册结果容易覆盖前面修改。

对 AI-Human，更推荐明确采用真正的 sequential reducer，或者采用结构化 Patch/Merge 契约，并定义字段冲突规则。

## 7. 完整 Agent Loop 中的扩展插槽

把 Registry、Hook 和 Middleware 放回 Agent Loop，关系如下：

~~~text
User Input
    │
    ▼
准备 Messages
    │
    ├── pre_llm_call
    │     动态 context 注入本轮 API view
    ▼
构造 Provider Request
    │
    ├── llm_request middleware
    │     修改最终请求 payload
    │
    ├── pre_api_request
    │     provider 请求边界扩展/观察
    ▼
llm_execution middleware
    │
    ▼
Real Provider
    │
    ├── post_api_request / api_request_error
    │
    ├── on_stream_start / delta / end
    │     异步观察，不修改主 token 流
    ▼
Parse Response
    │
    ├──────────── tool call ─────────────────────────────┐
    │                                                   │
    │                                      tool_request middleware
    │                                                   │
    │                                      pre_tool_call policy
    │                                                   │
    │                                      guardrail / approval
    │                                                   │
    │                                    tool_execution middleware
    │                                                   │
    │                                              Tool Handler
    │                                                   │
    │                                       post_tool_call observer
    │                                                   │
    │                                       transform_tool_result
    │                                                   │
    │                                result append 到 messages
    │                                                   │
    └────────────────── next LLM iteration ◄────────────┘
    │
    └──────────── final answer
                    │
                 pre_verify
              可要求继续一轮
                    │
            transform_llm_output
                    │
               post_llm_call
                    │
              finalize / output
~~~

这张图最值得记住的不是 Hook 名字，而是：

> Agent Loop 仍然是稳定主干，扩展能力被挂在请求边界、执行边界、安全边界和生命周期边界上。

## 8. Prompt 扩展：稳定规则与动态上下文必须分开

业务经常会说“我要加一段 prompt”，但这其实至少有两种完全不同的需求。

### 8.1 稳定 Prompt Section

例如：

~~~text
你是装备保障助手。
必须遵守某项业务规则。
本业务具有哪些固定能力。
~~~

Hermes 使用：

~~~python
ctx.register_system_prompt_section(...)
~~~

它的关键约束是：

- 在新 session 构造 system prompt 时冻结；
- section id 和位置受约束；
- content 有长度上限；
- 同一个 section id 不允许被另一个 Plugin 悄悄覆盖；
- 完整 system prompt 被核心持久化；
- session 恢复时按原字节恢复，不重新动态渲染。

这不是普通 prompt 拼接，而是在保护：

~~~text
一个 session 生命周期内
system prompt byte-stable
        ↓
prefix cache 可复用
~~~

### 8.2 动态 Context

例如：

~~~text
当前 tenant_id = xxx
当前 equipment_id = 123
当前业务阶段 = pending
本轮召回的业务信息 = ...
~~~

这些内容每轮都可能变化，不应该写入长期稳定的 system prompt。

Hermes 的 `pre_llm_call` 可以返回本轮 context，注入当前 user message 的 API view：

~~~text
用户真正发送的文字
        ≠
实际发给模型的 user content
~~~

概念上是：

~~~text
Visible User Message
       │
       ├── 用户可见、正常持久化
       │
       └── Model Context Projection
              + memory recall
              + plugin dynamic context
              + runtime-only information
~~~

Hermes 还会保存隐藏的 `api_content` sidecar，以便恢复时重现当时真正发给模型的字节和缓存前缀，同时不让用户恢复会话后看到这些内部注入内容。

这与 AI-Human 当前区分 Checkpointer 和用户可见 messages 的方向一致：

> 用户 transcript 和模型 projection 是两个视图，不要为了给模型补上下文而污染用户消息。

## 9. 三种“事件”不能混为一谈

Plugin 体系里最容易继续混乱的是 Event Bus、Stream Hook 和客户端 SSE Event。

### 9.1 Plugin Event Bus：进程内通信

Hermes 提供：

~~~python
ctx.emit("status_changed", payload)
ctx.subscribe("equipment:status_changed", callback)
~~~

它的语义是：

- emit 时强制使用 Plugin 自己的 namespace；
- `hermes:` namespace 保留给 Core；
- Plugin 可以订阅其他 namespace；
- host 使用单 worker 队列异步分发；
- 队列容量受限，满时丢弃新事件并 warning；
- subscriber 按注册顺序处理；
- 每个 callback 的失败被隔离；
- payload 深拷贝；
- 有递归深度保护；
- Plugin unload 时移除订阅，避免 zombie callback。

它解决的是：

~~~text
Plugin A ──内部事件──> Plugin B
~~~

它不是给浏览器或 App 的输出协议。

### 9.2 Stream Observer：观察模型 token 流

Hermes 的：

~~~text
on_stream_start
on_stream_delta
on_stream_end
~~~

是异步 Observer：

~~~text
Provider delta
     ├── 正常输出路径 → CLI / Gateway / TUI
     └── Plugin observer queue
             └── 独立 worker thread
~~~

每个 callback 有自己的 bounded queue，当前容量为 1024。慢 Plugin 不会阻塞 token 主路径；队列满时采用 drop-oldest。Reasoning delta 还需要显式 opt-in。

因此它适合：

- token metrics；
- streaming trace；
- 非关键旁路分析；
- 调试观察。

它不能：

- 修改 token；
- 阻止 token；
- 定义一种新的客户端 SSE 协议；
- 承担不可丢的业务事件。

### 9.3 Runtime Client Event：给用户界面看的结构化事件

这正是 Hermes 当前通用 Plugin API 的缺口。

Hermes 现有 `hermes.tool.progress` SSE 是在 `gateway/platforms/api_server.py` 中通过特殊 tagged tuple：

~~~text
("__tool_progress__", payload)
~~~

再由输出层硬编码为：

~~~text
event: hermes.tool.progress
~~~

当前通用 PluginContext 没有：

~~~python
ctx.emit_client_event(...)
ctx.register_sse_event(...)
~~~

所以如果一个 Plugin 想新增 `equipment.plan.updated`，现在仍需要修改输出协议或核心 gateway 代码。

对 AI-Human，这里不应照抄 Hermes。正确边界应是：

~~~text
Plugin
  │
  │ emit structured runtime event
  ▼
Runtime Event Sink
  ├── schema validation
  ├── run_id / message_id / block_id
  ├── monotonic seq
  ├── visibility
  ├── redaction
  ├── persistence
  ├── idempotence
  └── durable replay
          │
          ▼
Transport Adapter
  ├── SSE
  ├── WebSocket
  └── future transports
~~~

Plugin 应该调用类似：

~~~python
ctx.events.emit(
    type="equipment.plan.updated",
    payload={...},
    visibility="user",
    durability="durable",
)
~~~

而不应该：

~~~python
plugin.write_sse("event: equipment.plan.updated\n...")
~~~

因为让 Plugin 直接写 SSE 会绕开 Runtime 已有的顺序、落库、回放、脱敏和终态协议。

## 10. Plugin 生命周期：注册能力还必须能干净卸载

一个正式 Plugin 系统不能只解决“加载成功”，还必须解决：

~~~text
Plugin 加载一半失败怎么办？
reload 时旧 callback 怎么办？
Plugin 创建的后台 task 谁取消？
覆盖过的 provider / handler 谁恢复？
怎样防止卸载后还有 zombie callback？
~~~

Hermes 的重要做法是 ownership ledger：

~~~text
Plugin register(ctx)
    │
    ├── registration A ─┐
    ├── registration B  ├── ownership ledger
    ├── subscription C  │
    └── task D ─────────┘
              │
       unload / failed load
              │
       reverse-order teardown
~~~

具体原则包括：

- 每次注册返回或记录可撤销 handle；
- PluginContext 跟踪该 Plugin 拥有的资源；
- `register(ctx)` 中途失败时回滚已完成的部分注册；
- unload / force reload 时按反序注销；
- `spawn_task()` 创建的任务由 host 监督并在卸载时取消；
- replacement registry 恢复前一个值时，会检查当前槽位是否仍属于该 Plugin，避免覆盖后来注册者；
- event subscription 会随 owner 清除。

这是非常值得 AI-Human 直接吸收的部分：

> Plugin 能否 unload，是扩展边界是否真正封装好的验收标准。

还要明确一个安全事实：

> Hermes 原生 Plugin 是进程内 Python 代码。Capability consent 不是 sandbox；被加载的 Plugin 本质上拥有宿主进程权限。

如果 AI-Human 将来需要运行不受信任的第三方扩展，必须额外设计进程隔离、RPC、权限代理和资源限制，不能只靠 PluginContext 的 API 外观。

## 11. Hook / Middleware 的超时和失败策略

扩展点不是越多越好。每一个扩展点都必须定义：

~~~text
执行线程
顺序
返回值合并
timeout
异常传播
fail-open / fail-closed
是否允许修改数据
是否允许重入
是否可丢
~~~

Hermes 当前针对热路径 Hook 已有较细的策略：

- 一部分常用 turn hook 有 timeout；
- 普通 observer / transformer timeout 时通常 fail-open，跳过 Plugin，让 Agent 继续；
- `pre_tool_call` 属于安全 Policy，timeout / 未完成时 fail-closed，阻止工具执行；
- 某些 teardown、gateway policy、approval observer、kanban hook 有意不使用同一套 timeout wrapper；
- stream observer 不在线内执行，而是 bounded queue + 独立线程；
- callback 报错通常隔离，不让一个 Plugin 直接击穿整个 Runtime。

这说明超时策略不能按“所有 Hook 都 5 秒”简单统一：

~~~text
Observer 超时
  多数可以跳过

Security Policy 超时
  不能默认放行

Teardown flush 超时
  可能意味着状态丢失

Token observer 堵塞
  应移出主路径并允许降级/丢弃
~~~

不过 Hermes 当前仍有几个明显问题：

1. Middleware 没有独立 timeout，第三方代码仍可能卡住执行链。
2. 未知 Hook / Middleware 名称只 warning 但仍被保存，拼写错误可能变成永远不会触发的 dead registration。
3. `on_session_end` 实际更接近一次 `run_conversation` / turn 的结束，名字容易让开发者误认为整个长会话被销毁；真正 teardown 还有 `on_session_finalize` / `on_session_reset`。
4. `gateway/hooks.py` 还存在另一套基于目录和 `HOOK.yaml + handler.py` 的 Gateway HookRegistry，与 canonical PluginManager lifecycle hook 并存，属于重复扩展体系和历史债务。

## 12. 用 AI-Human 的五类真实需求验证这套边界

### 12.1 添加新工具

需求：某个业务增加 `query_equipment`。

正确路径：

~~~text
Plugin register_tool()
        ↓
Capability Registry
        ↓
按 app / tenant / session 解析
        ↓
当前 Agent Tool Snapshot
~~~

不应修改 Agent Loop，也不应让所有业务 session 自动看到它。

### 12.2 添加稳定业务规则

需求：装备业务长期遵守固定规则。

正确路径：

~~~text
register_system_prompt_section()
        ↓
创建 session 时冻结
        ↓
完整 prompt 持久化并稳定恢复
~~~

不能每轮重建，否则破坏 cache 和会话一致性。

### 12.3 添加动态业务上下文

需求：每轮注入 tenant、case、当前业务状态。

正确路径：

~~~text
pre-LLM context projection / typed context provider
        ↓
只修改本轮 Model View
        ↓
用户可见 transcript 保持干净
~~~

它不应伪装成用户真正输入，也不应改变稳定 system prompt。

### 12.4 修改模型或工具行为

需求：补工具参数、修改 provider request、fallback、cache、retry。

正确路径：

~~~text
补参数 / 改 payload
  → Request Middleware

fallback / cache / retry / trace
  → Execution Middleware

allow / deny / approval
  → Typed Policy
~~~

尤其不要把安全决策和普通 transform 混成同一个返回 dict。

### 12.5 新增 SSE / 前端业务事件

需求：新增 `business.plan.updated`。

正确路径：

~~~text
Plugin emits RuntimeEvent
        ↓
RunEventSink
        ↓
durable ordered event log
        ↓
SSE / WebSocket projection
~~~

SSE 是 transport，不是 Plugin API。

## 13. Hermes 值得学习的设计

### 13.1 窄腰核心，能力放在边缘

Hermes 把核心 Agent Loop 控制在相对稳定的主干中，让工具、provider、prompt、context engine、memory provider、platform 等从外部注册。

核心原则是：

> 产品可以在边缘快速增长，但每次模型调用都会承担的核心工具面和 prompt 前缀必须克制。

### 13.2 PluginContext 是受控入口

Plugin 不直接修改全局对象，而是通过统一 Context 注册。这使 host 能追踪 ownership、回滚和卸载。

### 13.3 请求修改发生在安全判断之前

`tool_request middleware → policy / approval → execution` 的顺序保证审批对象与真实执行对象一致。

### 13.4 Execution Middleware 保护至多一次执行

`next_call()` single-use，加上异常后的 downstream 结果保留，避免 provider 或副作用工具被意外执行两次。

### 13.5 Stable Prompt 与 Dynamic Context 分离

固定规则冻结进 session prompt；动态内容进入当前 Model View。这同时保护缓存、恢复和用户 transcript。

### 13.6 Stream Observer 离开 token 热路径

独立 bounded queue、独立 worker、慢消费者 drop-oldest，使非关键 Plugin 不拖慢用户看到 token 的速度。

### 13.7 Ownership Ledger 和失败回滚

Plugin 不是“import 完就不管”，而是 host 管理的资源集合，支持 partial registration rollback 和 reverse teardown。

### 13.8 扩展 payload 原生携带关联 ID

Plugin 可以根据 session / turn / request / tool call / task ID 建立 trace，不需要猜测调用关系。

## 14. Hermes 不值得照抄的部分

### 14.1 PluginContext 太宽

一个 Context 同时承载工具、provider、platform、prompt、hook、middleware、skill、event bus、state 和 task 等大量能力。它已经逐渐接近“受控但很大的宿主对象”。

AI-Human 可以使用 capability-scoped facade：

~~~text
ctx.tools
ctx.prompts
ctx.policies
ctx.middleware
ctx.events
ctx.lifecycle
~~~

再通过 manifest 权限决定 Plugin 能拿到哪些 facade。

### 14.2 Hook 语义混杂

Observer、Transformer、Policy、Dynamic Context 都挂在 Hook 名下，导致返回值、冲突和 timeout 规则越来越复杂。

### 14.3 Transform Hook 与 Middleware 重叠

`pre_tool_call modify` 与 `tool_request middleware` 都能改工具参数；`transform_tool_result`、`transform_llm_output` 也可以被建模为明确的 transformer pipeline。

### 14.4 Request Middleware 组合语义不够自然

多个 callback 看到同一初始 payload，再返回完整 replacement，不利于多 Plugin 安全组合。更适合使用顺序 reducer 或 typed patch。

### 14.5 没有通用客户端 Event Egress

Plugin Event Bus 和 Stream Observer 都不能替代 `RuntimeEventSink`。新增客户端事件仍要改 gateway 输出层。

### 14.6 多套 Hook 系统并存

canonical Plugin lifecycle hooks、Gateway filesystem hooks、shell hooks 的关系增加了理解和维护成本。新系统应尽量只有一份扩展类型系统和一份 lifecycle dispatcher。

## 15. 面向 AI-Human 的推荐骨架

结合 AI-Human 当前已经存在的 Runtime Event / SSE / replay 能力，更适合采用下面这套分层：

~~~text
                         AI-Human Runtime
                                │
                       ExtensionManager
                                │
       ┌────────────────────────┼────────────────────────┐
       │                        │                        │
Capability Registry      Lifecycle Observer       Pipeline Middleware
       │                        │                        │
 tools / prompts          trace / metrics          llm request
 providers / skills       audit / billing          llm execution
 schemas / event types    debug                    tool request
                                                    tool execution
       │                        │                        │
       └────────────────── Typed Policy ────────────────┘
                                │
                   Allow / Deny / Modify /
                       RequireApproval
                                │
                         Runtime Event Sink
                                │
                schema / seq / persist / replay
                                │
                      SSE / WebSocket / future
~~~

Plugin 只是把这些注册组合打包：

~~~python
class EquipmentPlugin:
    def register(self, ctx):
        ctx.tools.register(query_equipment)
        ctx.prompts.register_stable(equipment_rules)
        ctx.context.register_dynamic(current_equipment_context)
        ctx.policies.register_tool(equipment_guard)
        ctx.middleware.tool_request(inject_tenant)
        ctx.events.register_type(EquipmentPlanUpdated)
~~~

建议将扩展协议类型化：

~~~python
Observer:
    callback(event) -> None

Transformer[T]:
    transform(value: T, ctx) -> T

Policy[T]:
    evaluate(value: T, ctx)
        -> Allow | Deny | Modify[T] | RequireApproval

ExecutionMiddleware[Req, Resp]:
    invoke(request: Req, next_call) -> Resp

EventEmitter:
    emit(event: RuntimeEvent) -> EventEnvelope
~~~

同时从第一版就定义：

- registration order；
- dependency / priority；
- conflict resolution；
- timeout；
- fail-open / fail-closed；
- cancellation；
- reentrancy；
- at-most-once execution；
- unload rollback；
- capability permission；
- session snapshot；
- event durability 和 visibility。

## 16. Skills 和 MCP 在这张图中的位置

本章不深入 Skills 与 MCP，但要把它们放回正确位置。

### Skills

Skill 主要是给模型读取的操作知识和工作流说明：

~~~text
Skill
  → 教模型怎样使用现有 CLI / tools / files 完成任务
  → 通常不需要新增核心模型工具
~~~

Hermes 甚至把 skill slash command 作为 user message 注入，而不是修改 system prompt，以保护长期稳定前缀。

### MCP

MCP 主要是外部结构化工具协议：

~~~text
External MCP Server
        ↓ protocol
Hermes MCP Client
        ↓ tool schema / invocation
Agent Tool Surface
~~~

它适合把独立外部服务接成模型工具，但它解决的不是完整 Runtime 扩展：

- MCP 可以提供工具；
- 但不能天然覆盖 Agent Loop policy、prompt section、LLM middleware、runtime event sink 等宿主内部扩展点。

因此：

~~~text
Skill
  更像操作说明与工作流知识

MCP
  更像跨进程的外部 Tool Provider

Plugin
  更像宿主 Runtime 内的一揽子扩展安装包
~~~

三者不是竞争关系，而是不同层级。

## 17. 最终记忆模型

这一章最值得带走的不是 Hermes 具体有多少 Hook，而是下面这组边界：

~~~text
增加能力
  → Registry

观察事实
  → Observer

修改数据
  → Transformer / Request Middleware

作安全或流程决策
  → Typed Policy

包住真实执行
  → Execution Middleware

Plugin 之间通信
  → Internal Event Bus

向客户端输出业务状态
  → Runtime Event Sink

Plugin
  → 把以上能力打包注册，并接受宿主生命周期管理
~~~

一句话总结：

> Hermes 的 Plugin 设计真正有价值的地方，不是 `plugins.py` 提供了很多注册函数，而是它试图把不断变化的能力放到 Agent Runtime 的边界上；对 AI-Human 来说，下一步应进一步把 Observer、Policy、Transformer、Middleware 和 Runtime Event 明确类型化，使其他同事能够扩展业务，而无需进入核心 Agent Loop。

## 18. 主要代码入口

~~~text
hermes_cli/plugins.py
  PluginContext、PluginManager、registries、hook dispatch、event bus、ownership ledger

hermes_cli/middleware.py
  四类 middleware、request transform、execution onion chain、single-use next_call

agent/turn_context.py
  pre_llm_call 动态 context 注入

agent/conversation_loop.py
  llm_request、pre_api_request、llm_execution、post_api_request、pre_verify

model_tools.py
  tool_request、pre_tool_call、approval、tool_execution、post_tool_call、result transform

agent/turn_finalizer.py
  transform_llm_output、post_llm_call、turn 收尾 hooks

agent/plugin_stream_hooks.py
  stream observer 的 bounded queue 与 worker thread

gateway/platforms/api_server.py
  当前硬编码的 hermes.tool.progress SSE 投影

gateway/hooks.py
  独立的 legacy gateway filesystem hook system

agent/skill_commands.py / agent/skill_preprocessing.py
  skill 发现、slash command 和 prompt-cache-aware 注入

tools/mcp_tool.py
  MCP 工具连接与调用入口
~~~
