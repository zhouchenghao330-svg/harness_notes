# Hermes Agent 08：Memory / Context Engine / Compaction

> 状态：已共同阅读并确认。  
> 本章重点：区分 Hermes 的内置文件记忆、外部 Memory Provider、请求级 Context Selection 和有损 Compaction，并说明它们如何在 prompt cache、持久化和失败恢复约束下协作。

## 1. 核心结论

Hermes 这里不是一个笼统的“上下文管理模块”，而是三类职责不同的系统：

~~~text
Memory
  解决：跨 session 以后仍然要记得什么

Context Engine
  解决：当前 request 到底把哪些上下文交给模型

Compaction
  解决：当前 conversation 太长时，如何建立一个更短的继续边界
~~~

进一步拆开，模型可用的知识至少有四种来源：

~~~text
模型当前可用的上下文
├── Built-in Memory
│   └── MEMORY.md / USER.md 的 session-frozen snapshot
├── External Memory Provider
│   └── 根据本轮 user message 动态 recall 的相关记忆
├── Active Conversation Context
│   └── SessionDB 中 active=1 的当前 transcript 投影
└── Session Search
    └── 模型按需搜索完整历史、已归档 turn 和旧 session
~~~

必须避免把它们混成一个概念：

- Memory 保存长期稳定知识。
- SessionDB 保存完整、可审计、可恢复的历史。
- `select_context()` 只决定这一次 request 的上下文视图。
- Compaction 改变以后自动送入模型的 active context，但不会直接等于删除历史。

主要代码入口：

~~~text
tools/memory_tool.py
  内置 MEMORY.md / USER.md、memory tool、安全写入

agent/memory_provider.py
  外部 Memory Provider ABC

agent/memory_manager.py
  外部 provider 生命周期、recall、sync、checkpoint

agent/background_review.py
  后台 memory / skill review fork

agent/context_engine.py
  可替换 Context Engine ABC

agent/context_compressor.py
  内置 ContextCompressor 算法

agent/conversation_compression.py
  压缩事务、锁、checkpoint、DB commit、rotation

agent/native_compaction.py
  OpenAI Responses server-side compaction

agent/turn_context.py / agent/conversation_loop.py / agent/turn_finalizer.py
  recall、preflight、select_context、post-tool、post-turn 调用点
~~~

## 2. Hermes 有两套行为不同的 Memory

### 2.1 Built-in Memory：小型、受控、可编辑的策展区

内置 MemoryStore 管理两个文件：

~~~text
~/.hermes/memories/
├── MEMORY.md    Agent 学到的稳定事实、环境约定、工具特性
└── USER.md      用户身份、偏好、沟通方式和工作习惯
~~~

默认容量：

| Store | 默认限制 | 适合保存 |
| --- | ---: | --- |
| MEMORY.md | 2200 字符 | 稳定环境事实、项目约定、工具 quirks |
| USER.md | 1375 字符 | 用户身份、偏好、长期沟通习惯 |

这里按字符而不是 token 限制，因为字符数不依赖具体模型 tokenizer。

内置 memory 不应该保存：

- 当前任务进度；
- 临时 TODO；
- PR / issue / commit 编号；
- 一周后大概率过期的状态；
- 完整工作日志；
- 可复用操作流程。

对应去向是：

~~~text
短期任务状态       → 当前 conversation / todo
过去发生过什么     → SessionDB + session_search
长期稳定事实       → MEMORY.md / USER.md
可复用操作方法     → Skill
~~~

### 2.2 Frozen snapshot：持久化立即生效，prompt 不立即变化

MemoryStore 同时维护两个状态：

~~~text
MemoryStore
├── _system_prompt_snapshot
│   └── session 启动时冻结，用于 system prompt
└── memory_entries / user_entries
    └── live state，memory tool 修改后立即写盘
~~~

完整过程：

~~~text
session 启动
    │
    ▼
读取 MEMORY.md / USER.md
    │
    ├── 生成 frozen snapshot ──→ 进入 system prompt
    │
    └── 生成 live state
              │
          memory tool 写入
              │
              ▼
        磁盘文件立即更新
              │
              └── 当前 frozen snapshot 不更新
~~~

因此：

> memory 写入是立即 durable 的，但普通 turn 内不会修改已经缓存的 system prompt。

这样一次 `memory(add)` 不会让下一次模型请求的 system prefix 突然变化。

新的 memory 通常在下一次 session 启动时进入 prompt。Compaction 是例外：它本来就是允许重建上下文的边界，Hermes 会在这个边界重新加载 built-in memory。如果当前 cached prompt 已经逐字包含最新 memory block，则继续保留；否则才重建 prompt。

### 2.3 Memory tool

Hermes 只暴露一个 `memory` 工具，通过参数选择操作：

~~~text
memory
├── add
├── replace
├── remove
└── operations=[...]   一批操作原子提交
~~~

`replace` / `remove` 使用短的唯一子串定位条目，不要求模型维护额外 ID。

Memory 写入成功后返回 terminal result，并明确告诉模型不要重复操作。容量整理连续失败时，同一 turn 最多容忍三次；超过后要求模型停止 memory tool 循环、保持 memory 不变并继续回答用户。

### 2.4 Memory 文件安全

Memory 最终会进入 system prompt，所以不能把它当成普通文本文件。

~~~text
memory write
    │
    ├── injection / exfiltration pattern 扫描
    ├── 获取独立 .lock 文件锁
    ├── 锁内重新读取最新磁盘内容
    ├── 检查外部编辑导致的 round-trip drift
    ├── 检查字符容量
    └── temp file + atomic rename
~~~

关键不变量：

1. 文件读取失败不能当作空文件，否则下一次写入可能清空全部 memory。
2. replace/remove 前必须锁内重读，避免覆盖 sister session 的并发写入。
3. 检测到外部编辑产生的不可安全 round-trip 内容时，先生成 `.bak`，再拒绝覆盖。
4. 使用 atomic rename，读取者只会看到旧完整文件或新完整文件，不会看到写到一半的内容。
5. 写入时和 snapshot 构建时都扫描 prompt injection。
6. 可疑条目在 prompt snapshot 中变成 `[BLOCKED: ...]`，但原始文件仍保留，方便用户审计和删除。

## 3. External Memory Provider

### 3.1 Provider 生命周期

Honcho、Hindsight、Mem0、Supermemory 等外部记忆后端通过 `MemoryProvider` 接口接入：

~~~text
initialize()
system_prompt_block()
prefetch(query)
recall_status()
sync_turn(user, assistant)
queue_prefetch(query)
get_tool_schemas()
handle_tool_call()
on_pre_compress()
on_session_end()
on_session_switch()
shutdown()
~~~

当前 Agent 初始化路径只激活一个配置指定的 external provider，避免多个后端同时写入、召回结果冲突和 tool schema 膨胀。

### 3.2 一次正常 turn 的 recall / sync

~~~text
新 user message
      │
      ├── on_turn_start()
      │
      ├── 判断是不是 trivial prompt
      │       └── hi / thanks / slash command 等跳过 recall
      │
      ├── prefetch_all(user message)
      │       └── 获取本轮相关记忆
      │
      ├── recall 结果附加到当前 user API content
      │
      └── Agent loop
              │
              ▼
        turn 正常完成
              │
              ├── sync_all(user, assistant)
              └── queue_prefetch_all(user)
                     为下一轮预热
~~~

External recall 每个用户 turn 只在进入 tool loop 前获取一次，结果在该 turn 的多次模型调用中复用。

### 3.3 Recall 不修改干净 transcript

外部 recall 不进入 system prompt，也不直接写入用户可见 content：

~~~text
数据库 content：
    继续看看 08

模型收到的 api_content：
    继续看看 08

    [System note: recalled memory context...]
    ...
~~~

`api_content` sidecar 会保存当时真正发送给模型的精确字节。未来 replay 历史时使用 sidecar，而不是重新计算旧 recall，避免从 recall 注入点开始发生 prompt-prefix 漂移。

### 3.4 外部 provider 的隔离策略

- 单个 provider 失败不影响其他路径。
- External prefetch 有超时；超时后当前 turn 跳过。
- 上一条 provider prefetch 仍卡住时，不再创建第二条并发调用。
- turn 结束后的 sync/prefetch 在单 worker 后台队列里执行。
- 单 worker 保证 turn N 的写入先于 turn N+1。
- interrupted turn 不同步到外部 memory，因为部分回答和未完成工具链不是可信事实。
- shutdown 时只进行有界 drain，不能让坏 provider 无限阻塞进程退出。

## 4. Background Review

### 4.1 它是异步的

Background review 的模型调用和 memory/skill 写入，相对主用户流程是异步的。

~~~text
主 Agent 完成回答
      │
      ├── 持久化当前 turn
      ├── 判断是否达到 review cadence
      ├── 创建 daemon thread
      ├── thread.start()
      └── 主流程继续结束，不等待 review
                   │
                   ▼
            bg-review Agent
                   │
            memory / skill tool
                   │
              后台写入文件
~~~

线程启动本身发生在 turn finalization 中，但启动后没有 `join()`。因此用户可能已经收到回答，而 review 仍在后台调用模型。

Built-in memory 默认约每 10 个用户 turn 触发一次 review；也可以通过配置关闭 automatic background review，或使用 `/refine` 手动触发。

### 4.2 新 turn 到来时的短同步屏障

Review 不是完全放任运行：

~~~text
新用户消息到达
      │
      ▼
发现 bg-review 仍在运行
      │
      ├── 请求取消
      ├── 最多等待 2 秒确认 request phase 退出
      └── 不论是否确认，前台 turn 继续
~~~

所以准确结论是：

- review 工作异步；
- memory/skill 写入在 review thread 内同步执行；
- 相对主 Agent，它仍是后台异步写入；
- 下一个前台 turn 最多承担 2 秒取消确认等待；
- daemon thread 在进程直接退出时不保证完成，因此它属于 best-effort 自我维护。

### 4.3 Review fork 的隔离

Review fork 会继承父 Agent 的模型、认证信息、tool schema 和 cached system prompt，尽量命中相同 prompt cache；但它同时受到严格限制：

~~~text
Background Review Agent
├── 只允许 memory / skill management 工具
├── dangerous command 自动 deny
├── skip_memory=True，关闭 external memory provider
├── 复用父 Agent 的 built-in MemoryStore
├── _persist_disabled=True
├── 不写主 SessionDB transcript
├── 不运行 micro-compaction
└── 用户新 turn 到达时可取消
~~~

这里必须关闭 external provider，否则 review prompt 本身可能通过 `on_turn_start`、`prefetch`、`sync_turn` 被写进用户真实的外部记忆命名空间。

## 5. Context Engine

### 5.1 Context Engine 的职责

`ContextEngine` 不是单纯的 compressor interface，它允许替换整个上下文策略：

~~~text
ContextEngine
├── update_from_response()      接收真实 token usage
├── should_compress()           是否应该压缩
├── compress()                  怎样压缩
├── select_context()            本次 request 选择什么上下文
├── on_turn_complete()          观察并索引完成的 turn
├── on_session_start/end()
├── update_model()
├── get_tool_schemas()
└── handle_tool_call()
~~~

同一时刻只激活一个 Context Engine，由 `context.engine` 配置选择。默认是内置 `ContextCompressor`；外部 engine 加载失败时回退到内置 compressor。

### 5.2 `select_context()` 是 request-level projection

Hermes 同时区分：

~~~text
conversation_messages
    完整、权威、持久化的 transcript

api_messages
    本次真正准备发送给 provider 的消息
~~~

`select_context()` 位于两者之间：

~~~text
完整 conversation_messages
          │
          │ current user message + context budget
          ▼
ContextEngine.select_context()
          │
          ▼
本次 request 的 api_messages
          │
          ├── orphan tool-pair sanitize
          ├── role / thinking-only normalize
          ├── whitespace / JSON canonicalize
          └── prompt cache plan
          │
          ▼
        provider
~~~

它可以替换本次 request 的完整 message list，但不能修改持久化 transcript。

### 5.3 为什么 `compress()` 不够

两者解决的问题不同：

~~~text
compress()
  当前上下文太长
  → 建立一个更短、后续持续使用的压缩边界

select_context()
  当前问题属于哪个主题 / 分支
  → 只为这一次 request 选择相关上下文
~~~

例如一个 session 同时讨论过：

~~~text
A：部署架构
B：简历修改
C：Hermes Agent
~~~

用户重新问 Redis 部署问题时，检索式 Context Engine 可以构造：

~~~text
[system]
[A 中与 Redis 相关的历史]
[必要的最近状态]
[当前用户问题]
~~~

但 SessionDB 中的 A、B、C 都保持不变。下一轮用户切回简历，engine 可以重新选择 B。

这不是把 B/C 永久压缩掉，而是本次不发送它们。

### 5.4 为什么不能复用 memory recall

Memory prefetch 主要是追加：

~~~text
current user message + recalled knowledge
~~~

`select_context()` 则可以重构整个 request：

- 删除本轮无关历史；
- 检索相关 chunk；
- 选择 topic / branch；
- 从 DAG 展开需要的节点；
- 根据 budget 重新组合 message list。

因此：

~~~text
memory recall
    给当前请求补充额外知识

select_context
    决定当前请求整体由哪些上下文组成
~~~

### 5.5 为什么需要独立 hook

没有 `select_context()` 时，检索式 engine 为了每轮拿到 message list，只能滥用：

~~~python
def should_compress(...):
    return True
~~~

然后每轮通过 `compress()` 伪装成压缩，实际做 context routing。这会导致：

- 没有超限也进入 compaction 生命周期；
- selection 被 compression lock、cooldown、anti-thrash 影响；
- 宿主无法区分临时投影和有损历史改写；
- backend 暂时不可用可能被误判成 compression failure；
- 不必要地触发持久化和 session-boundary 逻辑。

所以 Hermes 把它拆成一对读写接口：

~~~text
select_context()
  turn/request 之前：从索引、DAG、topic state 读取并组装上下文

on_turn_complete()
  turn 完成之后：观察并索引刚完成的 transcript
~~~

### 5.6 `select_context()` 的安全边界

- Built-in ContextCompressor 不实现它，host 会识别基类 no-op 并完全跳过。
- 传给 engine 的 reference history 使用结构复制，防止插件通过嵌套 content/tool_calls 污染真实 transcript。
- 返回 `None` 表示使用原 request。
- 抛异常、返回空列表或非法值时 fail-open，继续使用原 `api_messages`。
- 返回结果仍经过统一 sanitizer，非法 tool pair 或 role sequence 不能直接到 provider。
- Prompt cache plan 在 selection 之后重新计算。
- 外部 engine 如果每轮选择不同 prefix，缓存损失由它自己的策略承担；默认路径保持 byte-stable。

如果系统只有线性 transcript + 超限 summary，不需要 `select_context()`。只有在做 topic isolation、DAG、branch、按问题检索历史等上下文虚拟化时，它才有明确价值。

## 6. Built-in ContextCompressor

### 6.1 触发入口

Hermes 不只在一个地方检查压缩：

~~~text
1. turn-start preflight
2. 完整 API request 组装后的 pre-API pressure check
3. tool result 加入以后
4. provider 返回 context overflow / 413 后的 reactive recovery
5. 手动 /compress
6. idle 后恢复时的 opt-in compaction
7. post-turn micro-compaction（opt-in）
~~~

默认配置：

| 配置 | 默认值 | 含义 |
| --- | ---: | --- |
| compression.enabled | true | 开启自动本地压缩 |
| threshold | 0.50 | 大窗口模型的比例触发点 |
| small-context floor | 0.75 | 小于 512K 时至少到 75% 才压缩 |
| max_attempts | 3 | 单 turn 最多压缩轮数 |
| tail_mode | lean | 使用较小 recent tail + 更强 summary |
| protect_first_n | 3 | 额外保护最前面的非 system message |
| protect_last_n | 20 | 至少保护最近 message 数 |
| in_place | true | 默认不旋转 session_id |
| proactive_prune_tokens | 0 | 默认关闭独立 tool-result prune |
| micro_compact | false | 默认关闭逐 turn 滚动压缩 |
| idle_compact_after_seconds | 0 | 默认关闭 idle 压缩 |

粗略 token estimate 故意偏保守，但 CJK 和 Responses reasoning replay 可能被高估。Hermes 会保存最近一次“rough estimate 与 provider real usage”的对应关系，用真实 usage 推算当前压力，避免长中文 session 在真实窗口只有 35%～55% 时过早压缩。

### 6.2 本地压缩算法

~~~text
当前 messages
      │
      ▼
Phase 1：机械清理 old tool results
      │
      ▼
确定 protected head / compressible middle / protected tail
      │
      ▼
summary model 总结 middle
      │
      ▼
组装 head + summary + tail
      │
      ├── 修复 role alternation
      ├── 修复 tool call/result 配对
      ├── 清除 stale media
      └── 清除 stale reasoning replay
      │
      ▼
检查是否真的变小
      │
      ▼
原子提交
~~~

Phase 1 不调用 LLM，包含：

1. 对 byte-identical tool result 去重，保留最新完整副本；
2. 将旧的大型 tool result 缩成摘要；
3. 截短旧 tool call 的超长 arguments；
4. 清理过时的图片 payload；
5. 保护 recent tail 和最新必要工具状态。

### 6.3 Head / Middle / Tail

~~~text
[ protected head ][ summarized middle ][ protected recent tail ]
~~~

- System prompt 隐式保护。
- 最前面的若干非 system message 保留。
- Lean mode 的 recent tail 约为 context window 的 2.5%，限制在约 10K～25K token。
- 仍需满足 `protect_last_n` 和最近真实 user-turn 锚点。
- 中间历史通过结构化 summary 压缩。
- 再次压缩时迭代更新已有 summary，而不是把旧 summary 当普通对话重复总结。

Lean summary 还会尽量保存：

- 用户原始要求；
- 当前 active task 和约束；
- tool chunk digest；
- 文件/命令等机械 anchor index；
- 可通过 `session_search` 恢复详情的指针。

### 6.4 Role 和 tool-pair 不变量

Summary 不能随便插入一个固定 role。

Hermes 会检查 summary 左右最近的 template-visible role，选择 `user` 或 `assistant`。如果无论选哪个都会产生同 role 相邻，则把 summary 合并进 tail carrier。

还必须确保压缩后：

- 至少存在一个有实际文本的 user turn；
- 不出现 orphaned tool result；
- tool_call_id 与 tool result 保持配对；
- summary 不被弱模型误认为新的用户要求或自己的历史回答；
- 旧 summary 的 synthetic scaffolding 不进入用户可见 transcript。

## 7. Proactive prune 与 Micro-compaction

### 7.1 Proactive tool-result prune

它是在完整压缩阈值之前执行的确定性清理：

~~~text
旧 tool output 很大
      │
      ▼
不等完整 compaction
      │
      ▼
仅机械清理 tool result
~~~

但它会重写 provider 已经见过的历史，因此会破坏 prompt cache prefix。Hermes 设置两道 hysteresis：

1. 回收 token 未达到最低门槛，不提交；
2. 提交后必须等待 message history 再增长一条完整 runway，才重新触发。

默认关闭。

### 7.2 Micro-compaction

Micro-compaction 在每个完整 turn 结束后，将最旧的一个完整 assistant/tool exchange 折叠进 rolling summary：

~~~text
turn 1 完成 → compact oldest exchange
turn 2 完成 → rolling summary + next exchange
turn 3 完成 → rolling summary + next exchange
~~~

好处是把一次大停顿摊到多个 turn；代价是每次都改写旧 prefix，可能每轮破坏 prompt cache。因此默认关闭。

Micro-compaction 特意不吸收 user message，用户原始指令保持 verbatim。它只吸收完整 assistant/tool exchange，不能切断工具组或留下连续 assistant role。

当 `checkpoint_required=true` 时，micro-compaction 会被强制关闭，因为它当前没有 pre-compress memory checkpoint seam。

## 8. Compression 事务与恢复

真正复杂的部分不在 summary prompt，而在提交边界。

~~~text
压缩请求
   │
   ├── 检查 cooldown / anti-thrash
   ├── 获取 session-scoped durable compression lock
   ├── 捕获 DB watermark
   ├── memory provider pre-compress checkpoint
   ├── deepcopy 原 transcript
   ├── 执行 summary
   ├── 验证结果
   ├── commit fence
   └── SessionDB 原子提交
~~~

### 8.1 Compression lock

同一 session 可能同时存在：

- 前台 Agent；
- gateway hygiene worker；
- background review fork；
- 多进程恢复路径。

如果它们在相同快照上同时压缩，可能生成两个 child 或覆盖彼此。因此 Hermes 使用 state.db-backed、按旧 session_id 加锁的 compression lease。

竞争失败不是“压缩无效”，而是临时 defer：当前调用保持原 messages，且不会消耗正常的 compression attempt budget。

### 8.2 Watermark

开始压缩时记录 message watermark。提交时只改写这个快照覆盖的历史，并保留 lease 期间其他 writer 新追加的 foreign tail，避免压缩完成时吃掉并发到达的新消息。

### 8.3 Commit fence 与 timeout

Summary 调用通过 streaming progress heartbeat 区分：

~~~text
模型很慢但仍持续输出 token
    → 延长 inactivity deadline

连接无输出、真正卡住
    → inactivity timeout

一直以极慢速度 trickle token
    → absolute total ceiling
~~~

如果 host 已经 timeout/cancel，commit fence 会阻止晚到的 summary worker 在调用者离开后继续修改 SessionDB。

## 9. Memory checkpoint：Memory 与 Compaction 的连接点

~~~text
准备丢弃旧 context
        │
        ▼
MemoryProvider.on_pre_compress(messages)
        │
        ├── 提取长期信息
        ├── 保存 checkpoint
        └── 返回 memory_context
                 │
                 ▼
       memory_context 进入 summary prompt
                 │
                 ▼
           才允许有损压缩
~~~

配置：

~~~yaml
compression:
  checkpoint_required: true
~~~

此时必须满足：

1. 存在 active external memory provider；
2. provider 声明支持 checkpoint API v2；
3. `on_pre_compress()` 成功完成。

任一条件不满足，压缩 fail closed，原 transcript 不动。

普通模式下，`on_pre_compress()` 是 best-effort；provider 返回的文本会先脱敏和限长，再传给支持 `memory_context` 的 Context Engine。

## 10. Summary 失败与 anti-thrash

### 10.1 必须 abort 的失败

以下情况保留原历史：

- 鉴权、权限、配额错误；
- 网络/连接失败；
- provider HTTP 成功但 summary content 为空；
- `abort_on_summary_failure=true`；
- required memory checkpoint 失败；
- summary 返回空 transcript；
- compression 语义上没有变化；
- commit fence 取消；
- DB 原子提交失败。

~~~text
summary abort
    │
    ├── messages 保持原样
    ├── session 不 rotation
    ├── 不丢 middle
    ├── 释放 compression lock
    └── 提示用户 /compress 或 /new
~~~

### 10.2 Static fallback

默认 `abort_on_summary_failure=false`。对于部分普通 summary failure，Hermes 会构造机械恢复摘要并丢弃 middle，使 session 尽量继续运行。

~~~text
fail closed
  保真度更高，但过大的 session 可能暂时无法继续

static fallback
  可用性更高，但被删除区域的信息保真度明显降低
~~~

对于高价值、强审计场景，可以考虑打开 `abort_on_summary_failure`，或使用 `checkpoint_required`。

### 10.3 Anti-growth

Commit 前使用同一种 rough token estimator 比较输入和输出：

~~~text
compressed_tokens > original_tokens
        │
        ├── 尝试机械 salvage
        └── 仍然更大 → 拒绝提交
~~~

这避免 summary、TODO snapshot、保留 reasoning 等组合起来反而让 transcript 变大。

### 10.4 Anti-thrash

Hermes 不只看“消息数有没有减少”，而是在下一次成功 provider response 中使用真实 `prompt_tokens` 验证：

~~~text
压缩完成
    │
    ▼
下一次 provider 返回 real prompt_tokens
    │
    ├── 已低于 threshold → 压缩有效，清除 strike
    └── 仍高于 threshold → ineffective strike
~~~

如果连续压缩无效、fallback streak 过高或 summary provider 处于 cooldown，自动压缩会暂时阻断，避免每个 turn 重复烧一次 summary 调用。手动 `/compress` 可以 force retry。

## 11. In-place 与 Rotation

### 11.1 默认 In-place compaction

~~~text
同一个 session_id
      │
      ├── 旧 active rows
      │       └── active=0, compacted=1
      │
      └── summary + tail
              └── 新 active rows
~~~

这是一个原子、非破坏性替换：

- 模型 resume 时只加载 active=1 的压缩后上下文；
- 压缩前 turn 仍保存在相同 session_id 下；
- 旧内容仍可被 FTS / `session_search` 搜索；
- title、cwd、goal、gateway routing 等不需要迁移；
- 用户仍然感知为同一个 conversation。

因此它非常接近“两套投影”的思想：

~~~text
完整 durable history
    用于 UI、审计、搜索、恢复

active model context
    用于下一次模型调用
~~~

### 11.2 Legacy rotation

`in_place=false` 时：

~~~text
parent session
      │ end_reason=compression
      ▼
child continuation session
      └── summary + tail
~~~

Rotation 需要迁移：

- goal；
- heartbeat；
- loop；
- title 和 title provenance；
- gateway route；
- logging/session context；
- memory provider session state；
- Context Engine lineage state。

这也是默认切到 in-place 的原因：session rotation 带来的 bug cluster 远大于它提供的价值。

## 12. OpenAI Responses Native Compaction

Hermes 也支持 provider-side compaction，但当前默认关闭，而且范围严格：

~~~text
codex_responses_native=true
+ gpt-5.6 family
+ api.openai.com 或 ChatGPT Codex backend
~~~

请求会附加：

~~~json
{
  "context_management": [
    {
      "type": "compaction",
      "compact_threshold": 200000
    }
  ]
}
~~~

服务端在超过阈值后返回 opaque encrypted `compaction` item。后续请求 replay 该 checkpoint，它就代表已被服务端压缩的旧历史。

~~~text
Responses native compaction
        │
        ├── 成功
        │     └── encrypted checkpoint 进入 codex_reasoning_items sidecar
        │
        └── provider 结构化拒绝 context_management
              ├── 当前 session 关闭 native
              ├── 去掉参数重试
              └── Hermes local compressor 继续作为 fallback owner
~~~

Native threshold 会被限制在 local compressor trigger 之前，确保 server 先获得压缩机会，但本地 compressor 始终保留兜底。

当 `checkpoint_required=true` 时，native compaction 被关闭。原因是 server 直接建立有损边界，Hermes 没有位置先调用 external memory provider 的 `on_pre_compress()`。

## 13. 设计评价

这一部分最值得学习的不是某一条 summary prompt，而是责任分离：

~~~text
Built-in Memory
  小、稳定、跨 session、进入 frozen system prompt

External Memory Provider
  动态 recall、自动 sync、可在压缩前 checkpoint

SessionDB
  保存完整、可审计的 durable transcript

select_context()
  构造单次 provider request 的临时上下文视图

Compaction
  改写 active model context，但保留原始历史用于恢复
~~~

可以概括成一句话：

> Hermes 没有让“模型当前看到什么”决定“系统永久保存什么”，也没有让“永久保存什么”意味着“每一轮都必须重新发送什么”。

这是 Memory、Context Engine、Compaction 和 Session Persistence 能够同时保持可扩展、可恢复和 cache-aware 的根本原因。

## 14. 源码定位

| 主题 | 文件 |
| --- | --- |
| Built-in MemoryStore | `tools/memory_tool.py` |
| Memory prompt guidance | `agent/prompt_builder.py` |
| Memory Provider ABC | `agent/memory_provider.py` |
| Provider 调度 | `agent/memory_manager.py` |
| Recall 注入与 api_content | `agent/turn_context.py`、`agent/conversation_loop.py` |
| Background review | `agent/background_review.py`、`agent/turn_finalizer.py` |
| Context Engine ABC | `agent/context_engine.py` |
| select_context 调用 | `agent/conversation_loop.py` |
| ContextCompressor | `agent/context_compressor.py` |
| Compression 事务 | `agent/conversation_compression.py` |
| SessionDB archive/rotation | `hermes_state.py` |
| Native Responses compaction | `agent/native_compaction.py` |
| Compression defaults | `hermes_cli/config_defaults.py` |

