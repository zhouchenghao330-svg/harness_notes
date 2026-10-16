# Hermes Agent 05：Tool System

> 状态：Tool System 框架已共同阅读并确认；Concrete Core Tools 正在按 `05_tools/` 逐组阅读。
>
> 本节结论：Hermes 的 Tool System 不只是“模型调用函数的 dispatcher”，而是一层 capability governance。它决定模型看见什么、session 被授予什么、调用何时能执行、多个调用如何排序，以及外部结果如何安全地重新进入上下文。

## 1. Tool System 解决什么问题

~~~
工具文件 / 插件 / MCP
        ↓ 注册与发现
Tool Registry
        ↓ session 选择
Toolsets
        ↓ 环境可用性过滤
check_fn
        ↓ Schema 动态化 / 清洗 / Tool Search
Provider tools：模型本轮真正看见的工具
        ↓
模型返回 tool_calls
        ↓
参数解析、授权范围、middleware、guardrail、审批
        ↓
执行调度与真实 handler
        ↓
结果治理、持久化、长度预算
        ↓
tool message 写回上下文
        ↓
下一次模型调用
~~~

工具系统至少要回答四个不同的问题：

| 问题 | Hermes 机制 | 说明 |
|---|---|---|
| 这个 session 被授予什么能力？ | toolset | session 能力边界 |
| 当前环境是否真的可用？ | 每个工具的 check_fn | Docker、浏览器、凭证、外部服务等 |
| 这一次调用是否被允许？ | approval、plugin hook、guardrail | 授权和运行策略 |
| 多个调用能否同时执行？ | batch segment planner | 并发与顺序 |

这些问题不能合并成一个 “tool available” 布尔值。

## 2. Registry：工具的注册表

Hermes 的工具文件通常在导入时调用 registry.register()，登记一条 ToolEntry。一个 ToolEntry 不只是 handler，还包括：

~~~
ToolEntry
├── name
├── toolset
├── schema
├── handler
├── check_fn
├── is_async
├── description / emoji
├── max_result_size_chars
└── dynamic_schema_overrides
~~~

Registry 负责：

- 发现并导入会自注册的内置工具模块；
- 保存全局内置工具；
- 保存按 HERMES_HOME/profile 隔离的插件工具覆盖层；
- 把工具名映射到 toolset；
- 生成 provider 使用的工具 schema；
- 将 handler 调用结果规范化为字符串或受支持的多模态 envelope；
- 将异常转换为统一的 tool error，而不是让整个 Agent Loop 直接崩溃。

Registry 有 generation 计数。注册、注销、MCP 动态刷新、toolset alias 变化都会递增 generation，使上层的工具定义缓存失效。

## 3. Toolset 与 check_fn：能力和可达性是两层

~~~
toolset
  = 这个 session 被授予哪些工具

check_fn
  = 这个已被授予的工具现在是否真的能工作
~~~

例如：

~~~
session 启用了 terminal/file toolset
        ↓
当前 Docker / SSH / Playwright / API credential 不可用
        ↓
对应 check_fn 失败
        ↓
工具 schema 不发送给模型
~~~

check_fn 结果有短 TTL 缓存，并对短暂探测失败保留最近一次成功结果的 grace window，避免一次 Docker 探测超时就让长会话突然失去工具。

但 check_fn 不能承担 session 身份判断：

~~~
错误方向：
  check_fn 判断“现在是不是桌面 GUI session”

正确方向：
  session platform / session toolset 决定 GUI 能力是否暴露
  check_fn 只判断 GUI bridge 是否可达、用户是否开启了功能
~~~

因为 check_fn 的结果可能被进程级缓存，而一个 Hermes 进程可以同时服务多个不同 session。

## 4. 工具如何进入模型请求

Hermes 最终发送给 provider 的是 OpenAI 风格的 function schema：

~~~json
{
  "type": "function",
  "function": {
    "name": "read_file",
    "description": "读取文件内容",
    "parameters": {
      "type": "object",
      "properties": {
        "path": {"type": "string"}
      },
      "required": ["path"]
    }
  }
}
~~~

工具定义生成的大致过程：

~~~
enabled_toolsets
        ↓
递归展开 toolsets / includes
        ↓
disabled_toolsets 做最终减法
        ↓
registry.get_definitions()
        ↓
check_fn 过滤不可用工具
        ↓
动态 Schema 重建
  - execute_code 可用子工具列表
  - Discord action / intent
  - delegate_task 当前限制
        ↓
Schema sanitizer
        ↓
Tool Search progressive disclosure
        ↓
provider tools
~~~

Hermes 会避免把“实际上不可用的工具名”继续写在其他工具描述中，防止模型看到 ghost vocabulary 后尝试调用不存在的工具。

## 5. Tool Search：工具过多时的渐进式暴露

工具 schema 会在每一次 API 请求中付出上下文和缓存成本。因此当 MCP/plugin 工具过多时，Hermes 不会把所有 schema 都直接交给模型：

~~~
核心 Hermes 工具
  └── 始终直接暴露

大量 MCP / plugin 工具
  └── tool_search
  └── tool_describe
  └── tool_call
          ↓
模型按需搜索、查看 schema、调用真实工具
~~~

Tool Search 只折叠非核心工具，并根据当前模型 context length 和配置决定是否激活。

执行 tool_call bridge 时，Hermes 会：

1. 解析出 underlying tool；
2. 用当前 session 的 enabled/disabled toolsets 重新做 scope 检查；
3. 对 underlying tool 的参数做 probe validation；
4. 之后所有 middleware、审批、guardrail、checkpoint 都观察真实工具名，而不是只看到 tool_call bridge。

这样可以避免受限 session 通过 bridge 绕过工具权限。

## 6. 一次工具调用的执行链

~~~
模型返回 tool_call
        ↓
解析 arguments
        ↓
Tool Search bridge 解包
        ↓
确认真实工具在当前 session scope 内
        ↓
Relay / request middleware
        ↓
plugin pre_tool_call
        ↓
guardrail before_call
        ↓
approval / edit approval
        ↓
显示 tool.started
        ↓
必要时创建文件 checkpoint
        ↓
真实 handler dispatch
        ↓
post_tool_call / result transform
        ↓
guardrail after_call
        ↓
结果长度治理与 prompt-injection 防护
        ↓
按 tool_call_id 生成 role=tool 消息
        ↓
增量写入 SessionDB
~~~

这里有一个重要的职责边界：

~~~
Registry：
  找到并运行 handler

Tool Executor：
  决定如何运行、是否允许、何时并发、如何写回结果

Agent Loop：
  决定拿到这些 tool result 后是否继续请求模型
~~~

## 7. 批量工具调用：按顺序组织并发段和屏障段

Hermes 不采用“这一批要么全并发、要么全串行”的粗粒度策略，而是把模型一次返回的多个 tool_call 切成连续 segments：

~~~
[ parallel: read_file(A), web_search(Q) ]
                   ↓ 等待全部完成
[ sequential: write_file(A) ]
                   ↓ 等待完成
[ sequential: read_file(A) ]
~~~

Planner 的核心规则：

- 明确列入 parallel-safe 白名单的工具，可以并发；
- MCP 工具只有明确 opt-in parallel 才可以并发；
- clarify 这种交互式工具永不并发；
- 无法解析参数的调用进入顺序屏障；
- 未知工具默认进入顺序屏障；
- read_file/search_files/write_file/patch 做路径重叠分析；
- reader 与 reader 重叠可以并发；
- 只要涉及 writer 的重叠，就关闭当前并发段；
- 后续调用不能越过之前的 barrier；
- 并发执行的结果最后仍按模型原始 tool-call 顺序写回。

路径冲突的意义是避免这种竞态：

~~~
write_file(A) 和 read_file(A)
~~~

如果两者同时跑，read_file 可能读到修改前或修改中的状态。因此 planner 会保证：

~~~
write_file(A) 完成
        ↓
read_file(A) 才能开始
~~~

这是一种保守但非常实用的 partial parallelism：安全的 I/O 尽量并发，副作用边界仍然严格。

## 8. 失败时到底怎么办：tool_call_id 如何保证配对

这是本节最容易被误解的地方。

假设模型返回：

~~~
call_1 = read_file(A)
call_2 = web_search(Q)
call_3 = write_file(A)
call_4 = read_file(A)
~~~

并且 planner 切成：

~~~
[parallel: call_1, call_2]
[sequential: call_3]
[sequential: call_4]
~~~

### 8.1 普通工具失败：默认继续后续调用

如果 write_file 的 handler 抛出普通异常，Registry 会捕获它并转换成一个 tool error 字符串：

~~~json
{
  "error": "Tool execution failed: ...",
  "tool": "write_file"
}
~~~

执行器仍然会为 call_3 生成：

~~~json
{
  "role": "tool",
  "tool_call_id": "call_3",
  "name": "write_file",
  "content": "{\"error\": \"...\"}"
}
~~~

然后默认继续执行 call_4。

所以在当前实现中，write_file 失败并不自动意味着 read_file(A) 被跳过：

~~~
write_file(A) 失败
        ↓
写入 call_3 对应的错误结果
        ↓
read_file(A) 继续执行
        ↓
写入 call_4 对应的读取结果
        ↓
所有结果一起交给模型
~~~

模型下一次看到的是：

~~~
call_3 → write_file 失败
call_4 → 当前文件实际状态
~~~

因此 read_file 读到的是“失败后的真实状态”。如果 write_file 在失败前已经造成了部分副作用，Hermes 不会因为普通异常自动回滚；checkpoint 为恢复/回退提供基础，但不是每次工具失败后的事务性 rollback。

这说明 Hermes 的 batch scheduler 解决的是：

~~~
调用顺序和竞态
~~~

而不是：

~~~
根据工具语义自动推导依赖图，并在前置失败后取消后续调用
~~~

它选择让模型一次看到完整结果，由模型决定下一步如何恢复。

### 8.2 为什么 ID 不会错

每个 tool_call 从模型返回时就有自己的 id。执行器在生成结果时始终使用当前调用的 canonical tool_call_id：

~~~
call_1 → tool_call_id=call_1
call_2 → tool_call_id=call_2
call_3 → tool_call_id=call_3
call_4 → tool_call_id=call_4
~~~

即使实际执行用了线程池，结果数组也按照原始索引收集并写回。这样 provider 看到的配对关系仍然是：

~~~
assistant.tool_calls[i].id
              ↕
tool.tool_call_id
~~~

并发改变的是物理完成时间，不改变协议顺序和配对关系。

### 8.3 哪些情况会停止后续调用

普通 handler failure 不会停止 batch，但以下情况会改变行为：

~~~
用户中断
  → 后续工具不再真正执行
  → 为剩余 tool_call_id 生成 cancelled / skipped result

KeyboardInterrupt / hard interrupt
  → 当前及剩余调用补齐取消结果
  → 避免 assistant tool_calls 没有对应 tool result

增量 SessionDB 持久化失败
  → 停止继续执行或继续向模型发送
  → 不把只存在于内存中的工具结果交给模型
  → 外层 turn 以 session_persistence_failed 结束

guardrail halt
  → 工具结果仍然会配对写回
  → 当前 batch 完成后，外层 Agent Loop 不再继续请求模型
~~~

中断时补齐 synthetic tool result 很重要：

~~~
assistant:
  call_3 = write_file
  call_4 = read_file

实际执行：
  call_3 被中断
  call_4 没有开始

仍然写入：
  tool(call_3) = cancelled
  tool(call_4) = skipped
~~~

否则下一次 provider 请求会出现一个只有 assistant.tool_calls、没有完整 tool results 的非法 transcript。

### 8.4 还有一道最终保险

在每一次 API 请求之前，Hermes 会执行 tool-call/result sanitizer：

~~~
发现没有对应 result 的 assistant tool_call
  → 注入 [Result unavailable] stub

发现没有对应 assistant call 的 tool result
  → 删除 orphan result

发现重复 tool_call_id
  → 去重

发现多个 id 表示法
  → 用 alias variants 进行匹配
~~~

因此 tool_call_id 的完整保障是三层：

~~~
执行器实时配对
        ↓
每个工具结果增量持久化
        ↓
下一次 API 请求前 sanitizer 最终修复
~~~

这也是为什么 Hermes 即使遇到进程中断、流式异常、恢复或跨进程 session 续接，也不会轻易把 tool-call transcript 弄成 provider 无法接受的状态。

## 9. 工具结果治理

工具执行结果不会原样无限制地进入上下文：

~~~
handler result
  ↓
异常统一为 tool_error
  ↓
web / browser / MCP 等外部内容
  → untrusted_tool_result 包裹
  → 中和伪造的 closing delimiter
  → 扫描潜在 prompt injection
  ↓
结果过大
  → 写入 HERMES_HOME/cache/spillover
  → 上下文只保留 preview + 路径
  ↓
单轮总结果超预算
  → 优先外溢最大的结果
  ↓
重复失败 / 重复无进展
  → guardrail 提醒、阻止或 halt
  ↓
按照原始 tool_call_id 写入上下文
~~~

工具结果的安全包装解决的是：

~~~
网页、MCP、浏览器返回的文本是数据，不是新的 system instruction
~~~

结果外溢解决的是：

~~~
工具输出不能无限膨胀上下文，并且大结果仍然可通过路径取回
~~~

Guardrail 解决的是：

~~~
Agent 不能对同一个失败调用或同一个无进展查询无限循环
~~~

## 10. 与 Pi 的差异

Pi 更倾向于把一个工具表达为完整的 ToolDefinition：

~~~
ToolDefinition
├── schema
├── prompt snippet / guidelines
├── prepareArguments
├── execute
├── executionMode
└── UI render
~~~

Hermes 则把长期运行 Agent 需要的治理职责拆成多个模块：

~~~
Registry
  发现、注册、schema、handler

Toolset
  session 能力边界

check_fn
  当前环境可达性

Tool Search
  工具渐进式暴露

Tool Executor
  审批、guardrail、checkpoint、调度、结果写回

SessionDB / sanitizer
  工具调用持久化与 transcript 修复
~~~

一句话总结：

> Pi 的 Tool System 更像“工具定义与执行框架”；Hermes 的 Tool System 更像“模型能力的注册、授权、调度、安全和持久化治理层”。

## 11. Concrete Core Tools 阅读入口

Tool System 框架完成不代表具体工具已经完成。按照 `all_harness/00_拆解共识.md` 的新规范，Hermes 另建：

~~~text
05_tools/
├── 00_inventory.md
├── 01_read_file.md
├── 02_write_file.md
├── 03_patch.md
├── 04_terminal_and_process.md
├── 05_search_files.md
├── 06_web_search_and_extract.md
├── 07_browser_tools.md
├── 08_execute_code.md
└── 09_clarify.md
~~~

前五组选择与 Pi Core Tools 可直接比较的文件和命令执行原语；后四组补充 Hermes 最有代表性的外部信息、Stateful Browser、Meta Tool 和 Human-in-the-loop 能力。仍然不把所有平台、业务、Plugin 和 MCP 工具扩展成产品功能清单。
