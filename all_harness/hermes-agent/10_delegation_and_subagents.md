# Hermes Agent 10：Delegation / Subagent / Background Run / MoA

> 状态：已共同阅读并确认。  
> 本章重点：Hermes 如何把一个拥有模型、工具和文件操作能力的自治执行单元，放进可控制、可观察、可恢复的执行边界；以及为什么 MoA 虽然也涉及多个模型，却不是普通 Subagent。  
> 补充来源：本章 MoA 部分同时吸收了用户在 `/side` 中与 Codex 讨论后确认的理解，尤其纠正 Aggregator 拓扑和 Consultation View 的两个常见误解。

## 1. 核心结论

Hermes 这里真正设计的不是“创建一个子 Agent”，而是一套小型多 Agent Runtime：

~~~text
                           父 Agent
                              │
                    delegate_task(tasks)
                              │
            ┌─────────────────┴─────────────────┐
            │                                   │
       控制面校验                           执行面准备
  goal / schema / depth / 权限         构造隔离的 Child Agent
            │                                   │
            └─────────────────┬─────────────────┘
                              │
                      创建 N 个子 Agent
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
           Child 1          Child 2          Child 3
          独立 loop         独立 loop         独立 loop
              │               │               │
      tool / reasoning  tool / reasoning  tool / reasoning
              └───────────────┼───────────────┘
                              │
                         Batch Barrier
                         等待全部结束
                              │
                    结果整理 / 限长 / 落盘
                              │
                ┌─────────────┴─────────────┐
                │                           │
          同步子委派                    顶层异步委派
       直接返回 orchestrator       durable ledger + completion queue
                                            │
                                      等父会话空闲
                                            │
                                  重新作为一个新 turn 注入
~~~

它重点解决的是：

- 子 Agent 应该知道哪些上下文，不应该知道哪些上下文；
- 子 Agent 能继承哪些工具，哪些能力必须强制剥离；
- 多个子 Agent 如何并行，而不把父上下文撑爆；
- 后台任务完成后，如何安全地回到正确会话；
- 子 Agent 卡死、失败、被取消、进程重启时，状态应该怎样表达；
- 父 Agent 如何查看、纠偏和终止正在运行的 child；
- MoA 如何在不改写核心 Agent Loop 的情况下加入多模型建议。

最值得记忆的设计原则是：

~~~text
1. Context 是显式复制的数据，不是共享可变历史。
2. Child capability 由 Runtime 推导，不由模型声明。
3. Async completion 是一个新事件 / 新 turn，不是修改旧 turn。
4. Long-running 与 stalled 是两种不同状态。
5. 可用结果与正常结束是两个不同维度。
6. Fan-out concurrency 必须配套 fan-in context backpressure。
7. Unknown 必须保持 unknown，不能被伪装成 clean / success。
8. 子 Agent 的总结是 self-report，不是 verified fact。
~~~

主要代码入口：

~~~text
tools/delegate_tool.py
  模型可见 delegate_task、child 构造、并行执行、控制面、结果整理

tools/async_delegation.py
  detached delegation、durable ledger、stale monitor、delivery claim

tools/process_registry.py
  completion queue、结果格式化、会话归属过滤

tools/subagent_worktree.py
  可选 git worktree 隔离与完成后的状态检查

tools/delegation_output_schema.py
  子 Agent 结构化输出约束、校验和一次修复

agent/subagent_lifecycle.py
  面向 Plugin 的稳定、不可变生命周期 API

agent/delegation_context.py
  delegated-child ContextVar、子进程环境和 dispatcher 身份隔离

agent/background_review.py
  memory / skill 自我改进使用的特殊后台 Agent fork

agent/moa_loop.py
  Reference fan-out、Consultation View、Aggregator facade、MoA accounting
~~~

## 2. 必须先区分的三套机制

### 2.1 Delegation / Subagent：多个执行者

Subagent 是一个完整 AIAgent：

- 拥有独立 Agent Loop；
- 可以调用被授权工具；
- 可以读写文件；
- 有独立 session、terminal task 和 iteration budget；
- 最终把摘要交还父 Agent。

它适合：

- 独立任务拆分；
- 并行代码调查；
- 大量中间信息的上下文隔离；
- 需要自己继续调用工具补充事实的任务。

### 2.2 MoA：多个只读顾问，一个执行者

MoA Reference Model 不是完整 Subagent：

- 没有 Hermes 工具；
- 不执行命令；
- 不读尚未进入 messages 的文件；
- 不写工作区；
- 只基于当前已有信息向 Aggregator 提建议。

~~~text
              当前 Agent 状态
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
     Advisor A   Advisor B   Advisor C
      只建议       只建议       只建议
         └───────────┼───────────┘
                     ▼
              Reference Guidance
                     │
                     ▼
            Aggregator / Acting Model
                     │
              正常 Hermes Agent Loop
                     │
              reasoning / tools / answer
~~~

因此：

~~~text
Subagent
  信息不够 → 自己调用工具继续调查

Reference
  基于已有信息 → 提供另一种判断
~~~

### 2.3 Background Review：特殊后台维护 fork

Background Review 也会创建 AIAgent，但它不是通用 delegation：

- 只允许 memory / skill 管理工具；
- 不把 review prompt 和 review transcript 写入用户会话；
- 直接修改 memory / skill store；
- 最后只向用户显示紧凑的改动摘要；
- 新前台 turn 到来时会优先取消 review。

它是一个权限收窄、持久化隔离、前台优先的后台维护进程。

## 3. 顶层异步，树内同步

当前模型可见的 `delegate_task` 不再让模型选择 `background` 和 `role`。Runtime 根据真实执行位置和配置决定。

~~~text
顶层 Parent Agent，depth = 0
    │
    ├── delegate_task
    │       └── 强制后台运行
    │
    └── 当前对话继续，不等待 child

Orchestrator Child，depth > 0
    │
    ├── delegate_task
    │       └── 同步等待自己的 workers
    │
    └── 拿到 worker 结果后综合，再返回上级
~~~

这解决了两个相反需求：

- 顶层 Agent 面向用户，不能因为一个长任务冻结整个聊天；
- Orchestrator 的职责就是等待 workers 并综合，不能在结果回来前结束。

当前默认 `MAX_DEPTH = 1`：

~~~text
Parent → Child
~~~

提高 `delegation.max_spawn_depth` 并开启 orchestrator 后，才允许：

~~~text
Parent
  └── Orchestrator
        ├── Worker A
        ├── Worker B
        └── Worker C
~~~

是否可以继续 delegate 由以下事实推导：

~~~text
child_depth = parent_depth + 1

orchestrator_ok =
    delegation.orchestrator_enabled
    AND child_depth < max_spawn_depth
~~~

调用者传入的 legacy `role` 只为旧协议兼容，不再决定真实能力。这样模型不能通过自称 `orchestrator` 获得额外权限。

## 4. 模型能控制什么，不能控制什么

当前模型主要能提交：

~~~text
tasks = [
  {
    goal,
    context?,
    output_schema?
  }
]
~~~

模型不能决定：

- child 的 `max_iterations`；
- 顶层任务是否 background；
- 是否获得 orchestrator 能力；
- 任意扩大 toolsets；
- trusted ACP command / args。

这些都由用户配置和 Runtime 控制，以免模型在一次 tool call 中临时提高自己的成本或权限。

Spawn 前会统一完成：

~~~text
goal 质量检查
tasks JSON 恢复与格式检查
batch 数量检查
output_schema 预校验
depth 检查
provider / credential 解析
工具继承和 deny 计算
child construction
~~~

结构化 schema 在任何 child 启动前统一校验。一个明显非法 schema 不会造成“部分 children 已经开跑，剩余任务才发现参数错误”的半提交状态。

## 5. Context Firewall：子 Agent 不继承父 transcript

普通 Delegation 没有把父 Agent 的完整 messages 复制给 child。

Child 得到的是：

~~~text
focused goal
+ explicit context
+ workspace / project instructions
+ concrete working directory
+ child-specific system prompt
~~~

Child 不会自动得到：

- 父 Agent 的完整历史讨论；
- 其他 child 的上下文；
- 与当前子任务无关的工具结果；
- 父 Agent 的 memory recall；
- 用户未显式交给它的信息。

~~~text
父上下文
├── user history
├── tool history
├── memory recall
├── 当前推理状态
└── 当前任务
          │
          │ 不是完整复制
          ▼
Child 上下文
├── focused goal
├── explicitly passed context
├── workspace instructions
└── allowed tools
~~~

这同时实现：

1. 降低 child 的 token 成本；
2. 减少无关历史干扰；
3. 缩小敏感上下文传播范围；
4. 强迫 Parent 把委派任务写成自包含 specification。

代价是 Parent 必须在每个 task 的 `context` 中重复传递必要的路径、约束、错误信息、语言和输出要求。

父上下文最终只看到：

~~~text
delegate_task call
+ dispatch acknowledgement
+ child final summary / completion event
~~~

不会自动吸收 child 的完整 reasoning 和中间 tool history。完整过程可通过 append-only live transcript 查看。

## 6. Capability Firewall：Child 只能缩权，不能扩权

工具继承原则：

~~~text
全局已注册工具
      │
Parent 当前 session 真正拥有的工具
      │
      ├── Parent 禁用的能力：不能获得
      ├── Child 请求但 Parent 没有：不能获得
      ├── composite toolset 展开后再次检查
      └── child blocked tools 强制移除
                    │
                    ▼
             Child 实际 Toolset
~~~

普通 child 默认禁止：

| 工具 | 禁止原因 |
| --- | --- |
| `delegate_task` | 默认禁止递归扩张；只有配置允许的深度派生 orchestrator 才能重新获得 |
| `clarify` | Child 不能绕过 Parent 直接向用户提问 |
| `memory` | 避免多个 child 并发修改共享长期记忆 |
| `send_message` | 避免以 Parent 身份产生跨平台外部副作用 |
| `cronjob` | 避免替 Parent 安排长期任务 |

blocked tool 不是只在 child 初始化时删一次，而是通过 disabled toolsets 保留 deny 约束，使动态 tool / MCP refresh 后也不能重新泄露。

### 6.1 危险命令审批

Child 运行在线程池中，无法使用 Parent CLI 当前线程的交互式 approval callback。若直接回退到 `input()`，会和拥有 stdin 的 prompt-toolkit 主线程死锁。

~~~text
Child 触发危险命令
          │
          ▼
delegation.subagent_auto_approve？
        /                      \
     false                      true
       │                         │
   自动 deny                 自动 approve once
   默认安全                  用户显式 opt-in
~~~

这和工具调度中的 `safe/unsafe` 不是同一层：这里讨论的是危险操作是否获得授权，而不是工具能否并行。

## 7. Child 构造时的隔离与继承

### 7.1 独立 Agent 状态

Child AIAgent 使用：

~~~text
quiet_mode = True
platform = "subagent"
ephemeral_system_prompt = focused child prompt
skip_context_files = True
skip_memory = True
clarify_callback = None
iteration_budget = fresh independent budget
~~~

这里的 `skip_context_files=True` 不表示不遵守项目说明。Parent 在构造 child prompt 时已经按统一逻辑读取必要 workspace context，再放进 focused prompt；这样避免 Child 自己重复构建一套可能不同的上下文。

### 7.2 Provider / credential 继承

默认 child 继承 Parent runtime。也可以通过 delegation 配置固定其他 provider / model。

同 provider 时，可以共享 credential pool 和 cooldown 状态；显式固定新 provider 时，则清除 Parent 的 fallback chain 和 OpenRouter routing filters：

~~~text
用户显式 pin delegation.provider
          │
          ├── 不允许静默回退到 Parent provider
          └── 固定 provider 失败时明确失败
~~~

这里选择的是 predictability over liveness：用户既然明确指定 child 必须走某个 provider，就不能在 quiet mode 中悄悄改道。

### 7.3 SessionDB connection 隔离

每个 Child 有独立 SessionDB connection，但指向 Parent profile 的同一数据库文件，并记录 `parent_session_id` lineage。

原因是 Parent 可能先关闭，而后台 child 仍需写入。若共用 Parent 的连接，Parent teardown 会把 child 正在使用的连接关闭，导致后续 transcript 静默丢失。

Child session 带 `_delegate_from` 标记，默认不出现在普通 session picker 中。

### 7.4 Terminal 与 cwd 隔离

每个 child 有独立 `task_id` 和 cwd 记录。它可以继承同一 terminal backend/container，但 child 的 `cd` 不会改变 Parent 的逻辑 cwd。

### 7.5 Process-global 状态保护

Child construction 会序列化，并在构造前后 snapshot / restore `model_tools._last_resolved_tool_names`。这是因为工具解析仍有一部分 process-global compatibility state；并发构造 Child 不能让它覆盖 Parent 的有效工具集合。

## 8. Delegation Context：防止继承错误身份

Hermes 进程可能本身就是一个 Kanban dispatcher worker，环境中存在 `HERMES_KANBAN_*`。

普通 delegated child 运行在同一 Python 进程，但它不是 dispatcher-owned worker。如果直接继承身份，它可能：

- 被强制加入 Kanban toolset；
- 收到错误 worker protocol；
- 把 `kanban_complete` 默认作用到 Parent task；
- 覆盖真正 worker 的结果。

Hermes 使用 `ContextVar` 标记：

~~~text
DELEGATED_CHILD_CONTEXT
NON_DISPATCHER_OWNED_CONTEXT
~~~

跨 subprocess 边界时，再清理 dispatcher-only env，并显式加入 delegated-child marker。

关键点是不能临时修改全局 `os.environ`：同一进程还有 Parent heartbeat、Gateway watcher 和其他并发任务正在读取它。ContextVar 负责进程内隔离，scrubbed env 负责跨子进程传播。

## 9. 同步 Batch：并行运行，按输入顺序汇总

Hermes 会先在主线程构造所有 child，再启动并行执行。

~~~text
tasks[0] ──→ Child 0 ──┐
tasks[1] ──→ Child 1 ──┼── 等待全部完成 ──→ results 按 task_index 排序
tasks[2] ──→ Child 2 ──┘
~~~

虽然 completion order 不确定，但最终结果按输入 task order 返回，避免模型根据线程完成顺序误配任务。

等待 futures 时使用短轮询，而不是一次不可中断的阻塞等待，这让 Parent interrupt 可以及时传播。

一个 child 失败不会让其他已运行 child 自动取消。所有任务各自形成结果条目，最终统一汇总：

~~~text
results = [
  {task_index, status, summary, error, ...},
  ...
]
~~~

整个 batch 只有一次 Parent 侧 finalize：统一进行 summary budget、memory hook、lifecycle hook、cost rollup 和 transcript manifest 更新。

## 10. Child Completion：结果可用性与退出原因分开

每个结果会记录：

~~~text
status
exit_reason
truncated
summary
error
api_calls
duration_seconds
tokens / cost
files_read / files_written
tool_trace
live_transcript
worktree state
~~~

其中：

~~~text
status
  回答“有没有可交付结果”

exit_reason
  回答“执行为什么结束”
~~~

因此可能出现：

~~~text
status = completed
exit_reason = max_iterations
truncated = true
~~~

意思是 Child 确实提供了可用 summary，但它不是正常完成全部任务，Parent 必须把它当成可能不完整的结果。

如果没有可用 summary，不能仅因为进程退出正常就标为 completed。

### 10.1 Structured output

若 task 带 `output_schema`：

~~~text
Child 首次回答
      │
      ▼
JSON / Schema 校验
   /             \
通过              失败
 │                 │
返回          仅把 validation errors
              交给 Child 修复一次
                       │
                  再校验并结束
~~~

这是一次有界 correction，不会无限要求模型修 JSON。第二次仍失败时，结果明确带 `schema_valid=false` 和 errors。

### 10.2 Self-report 不能直接当作事实

工具描述明确提醒 Parent：Child summary 是 self-report。

~~~text
Child 声称：文件已上传 / 远端写入成功 / 已发布
                     │
                     ▼
Parent 要求 URL / ID / absolute path
                     │
                     ▼
Parent 自己重新读取或查询验证
                     │
                     ▼
才能向用户确认成功
~~~

## 11. Fan-in Context Backpressure

Fan-out 只限制并发数量还不够。多个 child 同时返回长 summary，会在 fan-in 时撑爆 Parent context。

~~~text
10 children × 每个 30k 字符
            = 300k 字符一次性返回 Parent
~~~

Hermes 计算动态预算：

~~~text
Parent context length
- 当前 session_prompt_tokens
- Parent 输出预留 tokens
= 剩余 input headroom

剩余 headroom × 50%
÷ 有 summary 的 child 数量
= 每个 summary 动态预算
~~~

再与静态 `delegation.max_summary_chars` 上限取最小值。当前默认静态上限是 24,000 chars，最低动态预算为 2,000 chars。

超出预算时：

~~~text
完整 summary
├── 约 75% head：过程和主要发现
├── omitted middle：完整内容落盘
└── 约 25% tail：结论、改动和遗留问题
~~~

Parent 收到完整文件路径和建议的 `read_file offset/limit`，因此上下文被保护，但信息不必永久丢失。

这是一条非常通用的经验：

> 多 Agent 系统不仅需要限制 fan-out 计算量，还必须限制 fan-in 时跨越 Agent 边界的数据量。

## 12. 顶层 Background Delegation 的完整流程

顶层模型调用 `delegate_task` 时，当前 tool call 只得到 dispatch acknowledgement：

~~~text
assistant: delegate_task(tasks=[...])
tool: 已 dispatch；不要等待或轮询，继续当前工作
assistant: 可以继续处理其他内容或结束当前 turn
~~~

后台流程：

~~~text
Parent 调用 delegate_task
       │
       ├── 捕获 origin session / route / UI identity
       ├── 构造 children
       ├── durable persist：state=running
       ├── 从当前 foreground interrupt tree 脱离
       └── daemon executor 接管
                  │
          N 个 children 并行执行
                  │
             等待全部结束
                  │
          finalize + summary budget
                  │
        durable persist completion
                  │
           completion_queue
                  │
          owning session 空闲？
             /            \
           否              是
           │               │
        保留/重排队     claim delivery
                           │
                    新 user/internal turn
                           │
                    Parent Agent 被唤醒
~~~

整个 fan-out 是一个 background unit。N 个 children 全部结束后，产生一条 consolidated completion，而不是 N 条消息连续淹没对话。

若当前 surface 无法在原 turn 结束后接收 detached result，例如真正 stateless one-shot runner，则自动改成同步运行。API server 如果有可唤醒的 session id，可以在完成时 self-post 唤醒，仍允许后台执行。

若后台 pool 已满，新任务不会进入无界 backlog，而是拒绝异步调度并回退同步执行。

注意一个实现边界：同一个 `max_concurrent_children` 数字同时限制单个 batch 的 child 数和后台 delegation unit 数，但不等于严格的全局 child 总量。多个大 batch 同时存在时，总 child 数仍可能相乘。

## 13. 为什么 Completion 必须作为新 Turn 回流

后台结果不会：

- 回头修改原来的 tool result；
- 插进正在进行的 assistant/tool 序列中间；
- 直接修改正在执行的 AIAgent messages。

它会先进入共享 completion queue，等所属会话空闲后形成新的 synthetic user/internal turn。

~~~text
原始 turn
  assistant(tool_call delegate_task)
  tool(dispatch acknowledgement)
  assistant(继续当前工作 / final)

一段时间后
  synthetic user/internal:
    [ASYNC DELEGATION COMPLETE]
    Original goal: ...
    Original context: ...
    Status: ...
    Result: ...

  assistant:
    根据后台结果继续行动或回复用户
~~~

这样保住三个硬约束：

1. strict role alternation；
2. 已缓存历史前缀不被修改；
3. 已经结束的 tool-call protocol 不被重新打开。

Completion payload 是 self-contained 的，包含原 goal、context、model、dispatch/completion 时间、status、per-task summaries 和 live transcript。结果回来时 Parent 可能已经进入完全不同的话题，不能假设它还记得这项后台任务。

## 14. Durable Ledger 与投递协议

`tools/async_delegation.py` 在 profile `state.db` 中维护 `async_delegations` ledger。

主要信息包括：

~~~text
delegation_id
origin session / UI session / parent session
state
dispatch / completion time
task and result JSON
delivery_state / delivery_attempts / claim
owner PID / process start time
routing origin: scope_id / user_id / user_name
~~~

投递协议：

~~~text
persist completion
       │
       ▼
enqueue completion event
       │
       ▼
consumer atomic claim
       │
       ▼
尝试注入 owning session
     /                    \
成功                      失败
 │                         │
mark delivered          release claim
                           │
                        后续重试
~~~

关键边界：

- pending completion 最多尝试投递 8 次；
- 超过 48 小时的 completion 不再自动唤醒会话；
- delivered 等终态记录有保留和数量上限；
- stale claim 可以超时释放；
- completion 必须先 durable persist，再 enqueue；
- adapter 未接受消息前不能把 durable row 标为 delivered。

这不是数学意义上的分布式 exactly-once，但已经提供了持久化、认领、去重、失败释放和有界重试组成的 exactly-once-ish delivery。

### 14.1 Restart recovery

进程重启后：

~~~text
已完成但未投递
    → 从 ledger 恢复 completion event
    → 仍需正向证明 session ownership

仍标记 running/finalizing，但 owner PID 已死
    → status = unknown
    → 不自动重跑
    → 不伪造 success
~~~

Hermes 无法知道进程死亡前外部副作用是否已经发生。自动重跑可能重复发送消息、重复写远端数据或重复付费。因此 `unknown` 是唯一诚实状态。

### 14.2 Session ownership

多个 CLI/TUI/Desktop/Gateway session 共享同一 process queue，不能让最快醒来的 poller 随便拿走事件。

Ownership 可以通过：

- exact origin UI session；
- durable session key；
- Parent session id；
- compression lineage tip；
- Gateway 完整 route 和 tenant scope；

证明。

证明失败时 fail closed：事件保留给正确 owner 或停止自动注入，不会被另一个聊天窗口采用。

用户显式 `/new`、关闭 session 等边界不会被晚到 completion 重新复活；compression rotation 则仍属于同一逻辑会话，可以沿 lineage 交付到新 row。

## 15. Long-running 与 Stalled 分离

默认没有统一 child wall-clock timeout。持续运行很久不等于卡死。

Hermes 观察的 progress token 大致包括：

~~~text
api_call_count
current_tool
last_activity_ts
streaming activity
~~~

状态机：

~~~text
                    Child running
                          │
                   progress 改变？
                   /            \
                 是              否
                 │               │
          刷新 progress_ts    当前在工具中？
                               /        \
                             否          是
                             │           │
                         450 秒       1200 秒
                             └─────┬─────┘
                                   ▼
                               stalling
                                   │
                              请求 interrupt
                                   │
                           等待 120 秒 grace
                            /              \
                        正常退出           仍不退出
                           │                 │
                    保存部分结果       force finalize
                                             │
                                       status=stalled
~~~

只要 progress token 在变化，child 无论总共运行多久都不会被 stale monitor 处理。

若 force-finalize 后原 worker 又迟到，finalization claim 会拒绝第二次完成，避免同一个 delegation 产生两个相互冲突的 terminal event。

## 16. Live Control：List / Steer / Stop

### 16.1 List

只列出当前 Parent spawn tree 中的 live children。归属判断先看 object weakref ancestry；Parent AIAgent 对象被重建后，再用 durable session lineage 匹配。

这解决 `/model`、credential refresh、MoA one-shot 或 compression 后 Parent Python 对象变化，但后台 child 仍属于同一逻辑会话的问题。

### 16.2 Steer

Steer 不会切断当前 tool call，而是排队到下一次安全迭代边界：

~~~text
Child 正在执行 tool
          │
Parent 发出 steer
          │
          ▼
pending steer queue
          │
当前 tool 正常完成
          │
          ▼
tool result + steering text
          │
          ▼
Child 下一轮 reasoning
~~~

Acceptance 和 completion 通过 registry lock + `accepting_steer` 线性化：

~~~text
completion 先赢
    → steer 被拒绝

steer 先赢，但 Child 随后结束且没有下一个注入边界
    → 原 steering text 返回为 missed_steer
~~~

因此不会出现系统报告 steer 成功，但信息静默消失。

### 16.3 Stop

Stop 请求 hard interrupt，child 在下一 iteration boundary 停止，并尝试取消 in-flight tools。Python 线程不能被任意安全强杀，因此仍属于 cooperative cancellation；Child 已形成的 partial result 仍会通过正常 completion 交付。

## 17. Worktree Isolation

开启 `delegation.worktree_isolation` 且当前是 local git repository 时：

~~~text
Parent repository
├── parent working tree
├── .worktrees/subagent-a
├── .worktrees/subagent-b
└── .worktrees/subagent-c
~~~

每个 child 从 Parent 当前 HEAD 创建：

~~~text
branch = hermes-subagent/<id>
path   = <repo>/.worktrees/subagent-<id>
~~~

完成后 Parent 获得 branch、commits、dirty state 等信息，自行 review / merge。

若 child 没有 commit、也没有 dirty changes，可以自动 prune。若最终 inspection 本身失败，Hermes 不会声称 `0 commits / clean`，而是：

~~~text
inspection_failed
commits = UNKNOWN
dirty   = UNKNOWN
preserve worktree for manual inspection
~~~

这是 `unknown is not clean` 原则的具体实现。

边界：Worktree isolation 默认关闭，只支持本地 git backend；其他环境退化为共享 workspace。

## 18. Plugin-safe Subagent Lifecycle API

`agent/subagent_lifecycle.py` 为 Plugin 暴露稳定 API，但不暴露 live `AIAgent` 对象。

~~~text
SubagentLaunchRequest
SubagentHandle
SubagentStatus
SubagentTerminalState
SubagentCancelResult
SubagentResult
SubagentReconnectResult
~~~

插件只能通过 service：

~~~text
launch
status
wait
cancel
result
reconnect
~~~

监督 child，不能直接修改 Agent 内部状态。

Handle 带 HMAC capability，并校验：

- contract version；
- 字段类型；
- capability signature；
- 当前 active parent session ownership。

同一 Parent 下的 `correlation_id` 不能重复，避免插件意外启动重复工作。

生命周期状态包括：

~~~text
PENDING → STARTING → RUNNING
                    ├── SUCCEEDED
                    ├── FAILED
                    ├── INTERRUPTED
                    └── CANCEL_REQUESTED → CANCELLED

无法确认 → UNKNOWN
~~~

当前 public lifecycle 仍是 in-process registry。进程重启后 serialized handle 不能重新连接，`reconnect` 会明确返回 `RECONNECT_UNAVAILABLE`，而不是擅自重新启动任务。

## 19. Background Review：缓存与权限边界的特殊组合

Background Review 想同时做到：

1. 复用 Parent 已经热起来的 Prompt Cache；
2. 实际只能调用 memory / skill 工具；
3. 不能污染 Parent session；
4. 不能抢占下一次用户 turn。

### 19.1 Same model：完整 replay，复用 warm cache

默认使用 Parent 当前 model/provider/runtime，并尽可能保持：

- cached system prompt；
- effective ephemeral prompt；
- tools schema；
- reasoning config；
- prefill messages；
- provider routing pins；
- session id；

字节一致。

因此第一次 review request 可以复用 Parent 已经存在的长前缀缓存。

### 19.2 Different model：使用 digest

若用户把 review 路由到不同 model/provider，不可能复用 Parent 模型的 cache。此时重放完整 transcript 只会产生一次昂贵 cold write，因此 Hermes 改为发送机械生成的 conversation digest。

~~~text
same model      → full snapshot，追求 warm cache hit
different model → compact digest，减少 cold input
~~~

### 19.3 Schema parity 与 runtime whitelist 分离

Provider prompt cache 往往把 `tools[]` 也算入 key。若 Review Agent 只向模型暴露 memory/skill schema，就会从 tools 部分开始缓存 miss。

Hermes 采用：

~~~text
发给模型的 tools schema
    = 尽量与 Parent 保持字节一致
          │
          ▼
复用 Parent Prompt Cache

实际执行工具时
    = thread-local runtime whitelist
          │
          ▼
只允许 memory / skill
~~~

这是非常值得学习的分层：模型可见协议为了 cache parity 保持稳定，真正能力边界在执行面单独强制收窄。

### 19.4 共享 session identity，但隔离持久化

Review fork 为 cache parity 共享 Parent `session_id`，但设置：

~~~text
_persist_disabled = True
_session_db = None
_session_json_enabled = False
_end_session_on_close = False
~~~

否则它会把下面的维护对话写进用户真实 session：

~~~text
user: Review the conversation and update the skill library...
assistant: I will maintain memory and skills...
~~~

下一次恢复时，主 Agent 可能把后台 review prompt 当作用户的长期任务，发生 curator takeover。

Review 只允许通过工具修改 memory/skill store，不持久化自己的 harness transcript。

### 19.5 Detached in-memory compaction

Review 的第一条 request 保留完整 snapshot，以命中 warm cache。第一次 provider response 后，Review 可以只压缩自己的 in-memory transcript。

其 compressor 会先解除 Parent SessionDB / session id 绑定，并强制 in-place compaction。解除绑定失败时 fail closed：保持 compression disabled，避免任何 review 压缩状态写回 Parent session。

此外还有：

- 16 次 iteration 上限；
- 默认 600k aggregate input-token budget；
- 新 foreground turn 到来时请求取消；
- 取消确认只做有界等待，前台永远优先。

## 20. MoA 的角色与调用边界

这一部分最容易混淆。必须先区分五个角色：

~~~text
AIAgent / Agent Runtime
  Python 控制程序：维护 messages、执行工具、循环调用模型

Provider Client
  Runtime 请求模型使用的接口

MoA Facade
  伪装成 Provider Client 的本地中间层

Reference Models
  只读顾问：只给建议，不调用 Hermes 工具

Aggregator Model
  真正生成下一步 tool call 或 final answer 的 acting model
~~~

因此“主 Agent”和“主模型”不是同一个东西：

~~~text
AIAgent
  = Python runtime

Aggregator
  = 被 AIAgent 调用的 LLM
~~~

模型不会自己调用 Provider，也不会亲自执行工具。模型只返回结构化 tool call，Runtime 才真正执行。

## 21. 普通模式与 MoA Provider 模式

### 21.1 没有 MoA

~~~text
用户输入
   │
   ▼
AIAgent / Runtime
   │  整理 messages + tools
   ▼
真实 Provider Client
   │
   ▼
主模型 response
   │
   ├── tool_call → Runtime 执行工具，再调用模型
   └── final answer → 返回用户
~~~

代码形态近似：

~~~python
response = agent.client.chat.completions.create(
    messages=messages,
    tools=tools,
    stream=True,
)
~~~

### 21.2 MoA Provider 模式

原来：

~~~text
agent.client = RealProviderClient(...)
~~~

开启 MoA Provider 后：

~~~text
agent.client = MoAClient(...)
~~~

AIAgent 调用方式不变，但调用对象变成 MoA Facade：

~~~text
AIAgent / Runtime
        │
        │ messages + tools + stream + reasoning config
        ▼
     MoA Facade
        │
        ├── Reference A
        ├── Reference B
        └── Reference C
        │
        ▼
  收集 Reference Guidance
        │
        ▼
附加到原始 messages 尾部
        │
        ▼
Aggregator / Acting Model
        │
        ▼
标准 response / stream
        │
        ▼
AIAgent 按原 Agent Loop 处理
~~~

这里最重要的纠正是：

~~~text
错误理解：
References
    ↓
Aggregator 总结
    ↓
另一个 Main Model
    ↓
工具 / 回答

主要 MoA Provider 模式：
References
    ↓
Aggregator / Main Model
    ↓
tool call 或 final answer
~~~

Aggregator 就是这一次 Agent Loop 等待的主模型，不会再把它的结果交给另一个 acting model。

## 22. MoA Facade 拿到什么

Runtime 把原本准备发给主模型的请求交给 Facade：

~~~text
messages
tools
stream
temperature
max_tokens
reasoning config
extra body
...
~~~

Facade 从同一份状态生成两个不同视图：

~~~text
原始 messages
      │
      ├── Consultation View
      │       └── 给 References
      │
      └── 原始 messages + Guidance + 原始 tools
              └── 给 Aggregator
~~~

References 和 Aggregator 看到的内容、协议和能力都不同。

## 23. Consultation View 不是模型摘要

第二个需要明确纠正的误解：Consultation View 通常不是额外调用一个模型生成摘要，而是 `_reference_messages(messages)` 用代码机械转换。

原始消息：

~~~text
system:
  Hermes 大 system prompt

user:
  修复 auth.py

assistant:
  tool_call read_file({"path":"auth.py"})

tool:
  [大量文件内容]

assistant:
  tool_call search({"query":"timeout"})

tool:
  [大量搜索结果]
~~~

转换后：

~~~text
user:
  修复 auth.py

assistant:
  [called tool: read_file({"path":"auth.py"})]
  [tool result: 文件头部……
  ……中间省略……
  文件尾部]

assistant:
  [called tool: search({"query":"timeout"})]
  [tool result: 搜索结果头部……
  ……中间省略……
  搜索结果尾部]

user:
  请作为顾问判断当前状态、风险和下一步。
~~~

机械转换规则：

- 丢弃原 Hermes system prompt；
- 保留 user 文本；
- 保留 assistant 普通文本；
- 把真实 tool calls 展平为可读文字；
- 把 tool result 折叠进前一个 assistant turn；
- 大型 tool result 只保留 head + tail；
- 不发送真实 `tool` role；
- 不发送真实 `tool_calls` array；
- structured content 只提取可见文本，image-only turn 使用占位说明；
- 如果最后不是 user，追加一条 advisory instruction。

这样严格 Provider 不会因为 orphan tool result、未由自己产生的 tool call 或 trailing assistant prefill 而返回 400。

每个 Reference request 最前面还会加入专用 system prompt，明确：

- 你是只读顾问；
- 不能调用工具、终端、浏览器和文件；
- 不要假装执行过操作；
- 输出是交给 Aggregator 的 private guidance，不是用户答案。

## 24. Reference 有上下文，但没有环境访问能力

~~~text
有上下文
≠
能访问环境
~~~

Reference 能看到：

- 当前 user query；
- 已进入 messages 的历史 user / assistant 内容；
- Acting Agent 已经发出的 tool calls；
- 已经进入 messages 的 tool results 的受限预览；
- 当前任务已经暴露出来的进展。

Reference 看不到或不能取得：

- 尚未读取的文件；
- 尚未搜索的网页；
- 实时仓库状态；
- Hermes 完整工具 schema；
- 被截掉的 tool result 中间部分；
- 原 Hermes 大 system prompt。

因此，在 Acting Agent 尚未读任何代码时，Reference 只能建议“先查锁、先跑并发测试、先查 token refresh race”；它不能真的知道 bug 在哪。

MoA 更适合：

- 架构设计；
- 复杂方案权衡；
- 对已有证据的代码审查；
- 安全和风险分析；
- 检查 acting model 是否遗漏了方向。

它不适合代替 Subagent 搜集新事实。

## 25. 一次 MoA Agent Loop 时间线

假设初始状态：

~~~text
M0 =
[system]
[history]
[user: 修复 auth.py 的并发登录问题]
~~~

第一次调用：

~~~text
M0
 │
 ├── consultation_view(M0)
 │       ├── Ref A → 建议检查锁
 │       ├── Ref B → 建议检查 refresh token
 │       └── Ref C → 建议先写并发测试
 │
 └── 原始 M0 + Guidance G0
              │
              ▼
       Aggregator inference #1
              │
              ▼
       tool_call read_file(auth.py)
~~~

Facade 到这里完成这一次模型调用，把标准 tool-call response 返回 AIAgent。

Runtime 真正执行 `read_file`，然后 messages 变成：

~~~text
M1 =
M0
+ assistant tool_call read_file
+ tool result
~~~

Agent Loop 再次调用同一个 MoA Facade，Aggregator 产生下一次 tool call 或 final answer。

Facade 不执行工具，也不拥有完整 Agent Loop。它只把原来的一次模型 inference 增强成：

~~~text
先咨询 References
      ↓
再调用 Acting Aggregator
~~~

## 26. Fan-out Cadence：user_turn / per_iteration / every_n

### 26.1 `user_turn`：默认模式

References 每个新 user turn 只运行一次。后续 tool iterations 复用同一份 Guidance。

~~~text
User Query
    │
    ▼
References(M0) ─────────────────┐
    │                           │
    ▼                           │
Guidance G0                     │
    │                           │
    ▼                           │
Aggregator(M0 + G0)             │
    │                           │
    ▼                           │
read_file → Tool Result         │
    │                           │
    ▼                           │
Aggregator(M1 + 复用 G0) ◄──────┘
    │
    ▼
search → Tool Result
    │
    ▼
Aggregator(M2 + 复用 G0)
~~~

精确结论：

- References 能看到本 user turn 开始前已有的历史；
- References 能看到当前 user query；
- References 默认看不到这个 turn 后来才产生的 `read_file` / `search` 结果；
- 后续 Aggregator 能看到新 tool results，但仍复用 G0。

这是默认模式成本最低、延迟最小的原因。

### 26.2 `per_iteration`

~~~text
References(M0) → G0
Aggregator #1 → read_file

References(M1) → G1
Aggregator #2 → search

References(M2) → G2
Aggregator #3 → edit
~~~

References 能持续观察当前 turn 新产生的工具结果，但每次 Agent Loop inference 前都增加一轮 Reference fan-out，延迟和费用随 tool-loop 深度放大。

### 26.3 `every_n:N`

Iteration 1 一定刷新，然后每 N 个状态推进刷新一次，其余 iteration 复用最近 Guidance。计数器按 user turn 重置，冗余的同状态 retry 不消耗 cadence slot。

~~~text
user_turn      最便宜，但建议可能变旧
per_iteration  最新鲜，但最贵、最慢
every_n:N      两者之间的显式折中
~~~

Interrupted fan-out 的 placeholder 不会写入 reference cache，避免后续 iteration 永久复用一份残缺建议。

## 27. Aggregator 看见什么

Facade 构造 Guidance：

~~~text
[Mixture of Agents reference context]

Reference 1 — Model A:
建议检查 session refresh 的锁竞争……

Reference 2 — Model B:
建议先建立可重复的并发测试……

Reference 3 — Model C:
注意 token rotation 可能造成旧 token 覆盖新 token……
~~~

Aggregator 接收：

~~~text
原始完整 messages
+ Reference Guidance
+ 原始 tools schema
~~~

对比：

| 维度 | Reference | Aggregator |
| --- | --- | --- |
| 消息视图 | 拍扁、截断的 Consultation View | 原始 Agent messages + Guidance |
| 原 Hermes system prompt | 不接收 | 接收 |
| 真实 tool schema | 不接收 | 接收 |
| 调用 Hermes tools | 不能 | 可以返回 tool call |
| 最终面向用户 | 否，只提供 private advice | 是，作为 acting model |

因此 Aggregator 可以同时使用：

- Hermes 原始指令；
- 原始 conversation history；
- 当前真实 tool results；
- 多个 Reference 的建议；
- 自己的推理；
- Hermes 工具。

## 28. Guidance 为什么必须放在尾部

本轮 Guidance 可能变化。如果回头修改早期 user query：

~~~text
System
User Query + 新 Guidance       ← 很早开始变化
Assistant Tool Call 1
Tool Result 1
Assistant Tool Call 2
Tool Result 2
~~~

Provider KV cache 只能复用到变化点之前，后面的长历史必须重新 prefill。

Hermes 改为：

~~~text
System
User Query
Assistant Tool Call 1
Tool Result 1
Assistant Tool Call 2
Tool Result 2
Reference Guidance             ← 尾部变化
~~~

这样稳定前缀仍然可以命中缓存，Guidance 又具有最高 recency。

附加规则：

- 最后一条已经是 user：合并进这个尾部 user content；
- content 是 structured list：追加新的 text part；
- 最后一条不是 user：追加一条新的 trailing user guidance；
- failover 重新规划 cache breakpoint 前，先 `peel_reference_guidance()` 还原 base transcript，再重新装配。

这里遵守的是 Hermes 的总原则：每轮变化的数据尽量只出现在稳定前缀之后。

## 29. MoA Facade 为什么是优秀的 Narrow Waist

MoAClient 实现与普通 Provider Client 相同的接口：

~~~text
client.chat.completions.create(...)
~~~

对 AIAgent 来说，内部究竟是：

~~~text
一次真实 Provider 请求
~~~

还是：

~~~text
N 次 Reference 请求
+ 1 次 Aggregator 请求
~~~

都返回标准 response / stream。

因此以下逻辑仍由原 AIAgent 统一负责：

- streaming consumer；
- reasoning 解析；
- tool-call parsing；
- tool execution；
- stale-stream / retry / repair；
- iteration gate；
- finalization 和 persistence。

MoA 只扩展“一次模型调用的内部实现”，没有复制完整 Agent Loop。这就是 Facade Pattern 在 Agent Harness 中非常漂亮的用法。

## 30. MoA 的失败、取消、费用与隐私

### 30.1 Reference failure degradation

单个 Reference 失败会变成内部 labelled sentinel，并从真实 Guidance 中过滤。其他 References 和 Aggregator 仍继续。

所有 References 都失败时：

~~~text
跳过无意义的 synthesis
      ↓
Aggregator acting alone
      ↓
按 degraded_reference_policy 决定是否提示降级
~~~

不会因为一个辅助顾问失败而让正常 Agent Loop 整体不可用。

### 30.2 Interrupt

Reference futures 使用短轮询观察 `agent._interrupt_requested`。尚未启动的 future 可以 cancel；已经进入 blocking HTTP call 的线程不能强杀，只能停止等待。

已在 Provider 运行的请求即使结果被 abandon，仍可能计费。Hermes 为这些 late futures 注册 accounting callback，在请求最终结束后把真实 token/cost 补入待消费统计，避免“用户中断后费用消失”。

### 30.3 独立 usage / cost accounting

每个 Reference 可能使用不同 provider/model，token wire format 和价格都不同。Hermes 先按每个 Reference 自己的 runtime 标准化 usage、按自己的模型价格估算 cost，再汇总美元金额。

不能把 Reference token 全部按 Aggregator 的价格重算。Cache hit iteration 也不能重复计费同一轮 advice。

### 30.4 Privacy filter

Advisor 输出可能回显 conversation 中的邮箱、电话或 credential。Hermes 区分：

~~~text
display
  用户可见 Reference blocks 和持久化 traces 脱敏
  Aggregator 仍接收原始 advice，保留推理质量

full
  除 display/traces 外，交给 Aggregator 的 Guidance 也脱敏
~~~

Raw reference cache 保留在内存中，脱敏发生在具体消费面，避免重复脱敏或配置切换时泄漏。

MoA trace 会保存 Reference 输入输出、Aggregator 实际输入输出和各 slot metrics；tracing 失败不影响用户 turn。

## 31. Hermes 源码中存在两种 MoA 拓扑

这一点必须诚实记录，因为它正是讨论中最容易绕晕的原因。

### 31.1 主要 MoA Provider / MoAClient 模式

~~~text
References
    ↓
Aggregator / Acting Main Model
    ↓
标准 response / tool call
    ↓
外层 AIAgent Runtime
~~~

Aggregator 就是主模型。

### 31.2 `aggregate_moa_context()` 旧式 / 显式 context 路径

代码中仍有另一条拓扑：

~~~text
References
    ↓
Synthesis Aggregator
    ↓
MoA Context
    ↓
另一个 Main Model
    ↓
tool call / final answer
~~~

所以判断代码时要看入口：

~~~text
MoAClient / provider="moa"
  → Aggregator 就是 acting model

aggregate_moa_context / 显式 moa_config 路径
  → Aggregator 先生成 context，之后还有主模型调用
~~~

当前主要虚拟 Provider 模式采用第一种。不能用第二种拓扑概括所有 Hermes MoA 行为。

## 32. Delegation 与 MoA 的选择

| 问题 | Delegation / Subagent | MoA Reference |
| --- | --- | --- |
| 是否是完整 AIAgent | 是 | 否，只是独立模型请求 |
| 是否拥有工具 | 受限工具集 | 无 Hermes 工具 |
| 是否能搜集新事实 | 可以 | 不可以 |
| 是否有独立 session/terminal | 有 | 没有 |
| 是否适合并行调查 | 是 | 只适合并行观点 |
| 中间过程是否进入 Parent context | 不进入，只回 summary | Guidance 进入 Aggregator prompt |
| 主要价值 | 分工、隔离、并行执行 | 多视角判断、风险检查、方案权衡 |

选择原则：

~~~text
需要自己继续读文件、搜索、执行命令
    → Subagent

已有足够证据，需要不同模型从多个角度审视
    → MoA

只是机械多步操作，没有独立推理价值
    → Parent 直接执行或 execute_code
~~~

## 33. 设计中最值得学习的地方

### 33.1 控制面和数据面分开

~~~text
控制面
  id / state / ownership / cancel / steer / delivery claim

数据面
  goal / context / tool results / summary / guidance / transcript
~~~

控制面保持小而结构化，完整大文本放在 transcript、spill file 和 completion payload 中。

### 33.2 权限由事实推导

Child 是否可以继续 delegation 取决于 depth/config；是否可以控制某 child 取决于 object ancestry 和 durable session lineage。模型声明、随机 id 和当前 Python 对象都不能单独构成权限。

### 33.3 Async 结果通过事件回流

异步任务不是直接触碰正在运行的 Agent 内部状态，而是持久化 completion，等会话进入安全边界后重新触发一个 turn。

### 33.4 状态表达诚实

~~~text
owner process died      → unknown
worktree inspect failed → inspection_failed / unknown
hit max iterations      → completed + truncated
steer 未送达            → missed_steer
schema 修复后仍失败      → schema_valid=false
~~~

系统不会为了得到一个简单状态而掩盖不确定性。

### 33.5 缓存优化不能替代安全边界

Background Review 为了 cache parity 保持 prompt/tools schema，但通过 runtime whitelist 强制权限；MoA 把 Guidance 放在尾部，但 Aggregator 仍走正常工具授权和 Agent Loop。Cache optimization 和 capability enforcement 是两套独立机制。

## 34. 当前实现边界与批判性观察

- `tools/delegate_tool.py` 接近五千行，已经有成为新 god-file 的风险；child build、control plane、result shaping、batch dispatch 可以继续模块化。
- Child 主要运行在同进程线程，不是强进程或容器安全隔离。
- Blocking Provider HTTP call 不能真正 kill；interrupt 后请求可能继续执行和计费。
- Worktree isolation 默认关闭，共享 workspace 中的多个 child 仍可能产生写冲突。
- Public lifecycle registry 只在当前进程存活，不能恢复进程重启前的 running child。
- `max_concurrent_children` 同时用于单 batch parallelism 和后台 unit capacity，但不是严格全局 child 上限。
- MoA Reference 没有工具，副作用风险小，但它的建议仍可能误导 Aggregator；privacy filter 不等于 trust boundary。
- 源码中有少量 stale comment，例如个别位置仍写旧 default depth / concurrency，判断行为应以实际常量、配置读取和 dispatch path 为准。
- Durable delivery 是 exactly-once-ish，而不是跨数据库和外部消息平台的严格事务性 exactly-once。

## 35. 一句话总结

> Hermes Delegation 不是简单的“开几个子 Agent”：它围绕不可靠、可并发、可能越权和可能失联的执行者，建立了上下文防火墙、能力防火墙、生命周期控制、结果背压、异步持久化回流和故障诚实性；而 MoA 则通过 Provider Facade 把多个无工具顾问嵌入一次模型调用，最终由 Aggregator 这个 acting main model 继续原有 Agent Loop。
