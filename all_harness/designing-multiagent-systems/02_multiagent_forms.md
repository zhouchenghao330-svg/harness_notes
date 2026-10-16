# 组件 02：Workflow、Orchestration 与 Agent-as-Tool

> 状态：已共同阅读
>
> 阅读重点：把仓库当作概念和设计模式样本；代码用于确认边界，不把实现细节本身当作唯一正确答案。

## 1. 本组件要回答的问题

- 三种形态分别把控制权放在哪里？
- “步骤中使用了 Agent”和“自治多智能体”有什么区别？
- 什么时候应该使用确定性 Workflow？
- 什么时候应该让调度器或 Coordinator Agent 做动态决策？
- 三种形态的上下文、状态、错误和成本边界分别是什么？

## 2. 主要代码入口

- `picoagents/src/picoagents/workflow/`
- `picoagents/src/picoagents/orchestration/`
- `picoagents/src/picoagents/agents/_agent_as_tool.py`
- `examples/workflows/`
- `examples/orchestration/`
- `examples/agents/agent_as_tool.py`

## 3. 共同阅读结论

三种形态的根本区别不是“系统里有几个 Agent”，而是：

> 多个 Agent 外面的那一层控制循环由谁掌握，以及一次执行的结果以什么形式跨越边界。

单 Agent 内部已经存在自己的运行循环：

~~~text
模型调用
  ↓
文本或 Tool Call
  ↓
执行工具
  ↓
Tool Result
  ↓
模型继续或结束
~~~

多智能体系统在它外面再增加一层组合控制：

~~~text
选择一个执行者
  ↓
执行这个 Agent 自己的内部 Loop
  ↓
取得结果
  ↓
决定结果如何进入全局状态
  ↓
决定下一步
~~~

三种形态分别把这层控制放在：

| 形态 | 外层控制者 | 结果跨边界的形态 |
|---|---|---|
| Workflow | 开发者定义的图 + WorkflowRunner | 下游 Step 的结构化输入 |
| Orchestration | Orchestrator 的持续运行循环 | 团队共享 transcript / state 的新增内容 |
| Agent-as-Tool | 上层 Coordinator Agent 的模型 | 上层 Agent 收到的 ToolResult / ToolMessage |

## 4. Workflow：图在调用 Agent

### 4.1 核心模型

Workflow 先由开发者定义 Step、Edge、起点、终点和数据类型：

~~~text
ResearchStep
     ↓
WriteStep
     ↓
ReviewStep
     ↓
结束
~~~

运行时不是某个 Agent 自己选择下一位 Agent，而是 `WorkflowRunner` 根据图结构和已经完成的依赖计算 ready steps。

因此控制关系是：

~~~text
开发者定义图
     ↓
WorkflowRunner 解释图
     ↓
Step 被启动
     ↓
如果 Step 包装了 Agent，再调用 agent.run()
~~~

### 4.2 Agent 在 Workflow 中的身份

`PicoAgentStep` 把一个完整 Agent Runtime 包装成普通 Workflow Step：

~~~text
PicoAgentInput
  task
  additional_context
      ↓
PicoAgentStep.execute()
      ↓
agent.run(task)
      ↓
PicoAgentOutput
  response
  messages
  usage
  metadata
~~~

这说明：

> Workflow 中使用了多个 Agent，不等于这些 Agent 拥有全局自治权。

Agent 只负责完成当前节点任务；下一步是否合法、何时启动以及结果交给谁，仍由图控制。

### 4.3 状态形态

Workflow 主要维护结构化执行状态：

~~~text
WorkflowExecution
├── state
├── step_executions
│   ├── research: completed
│   ├── write: running
│   └── review: pending
└── error
~~~

核心数据流是：

~~~text
上游 Step 的 Output
          ↓
下游 Step 的 Input
~~~

当前实现还会比较上下游 Pydantic Schema，并在运行前检查：

- 是否存在起点和终点；
- Edge 引用的 Step 是否存在；
- 是否存在 cycle；
- 是否存在不可达 Step；
- 相邻 Step 的输入输出类型是否兼容。

因此 Workflow 是 **dataflow-centric**：中心问题是数据如何沿确定性结构流动。

### 4.4 理论优势和代价

优势：

- 路径清楚，容易测试；
- 输入输出可以类型化；
- 容易定位失败步骤；
- 适合 checkpoint、恢复和审计；
- 并发、分支和汇合可以显式表达；
- 可以把权限和人工审批放在明确节点。

代价：

- 开发者必须提前知道大部分流程；
- 对运行时出现的新情况适应较弱；
- 如果把大量语义判断硬编码成条件边，图可能快速膨胀；
- 一个错误的静态流程会稳定地重复错误。

### 4.5 适用问题

当下面这些条件成立时优先考虑 Workflow：

- 任务阶段相对稳定；
- 失败和恢复边界很重要；
- 上下游能够定义清晰合同；
- 希望控制成本和最大执行范围；
- 需要可靠地插入人工审批或外部事务。

## 5. Orchestration：调度器在持续运行多个 Agent

### 5.1 核心模型

Orchestration 不是按 DAG 一次经过若干节点，而是在一个持续循环中反复运行 Agent：

~~~text
select next agent
  ↓
prepare context for agent
  ↓
run agent
  ↓
update shared state
  ↓
check termination
  └── 未终止则进入下一轮
~~~

本仓库的 `BaseOrchestrator` 把以下内容作为公共运行时状态：

- `agents`
- `shared_messages`
- `iteration_count`
- `termination`
- `max_iterations`
- 各模式自己的 metadata

因此 Orchestration 更接近一场持续团队协作：上一位 Agent 的结果进入共享状态，下一轮再决定由谁继续。

### 5.2 Orchestration 不等于“编排智能体”

必须分开两个问题：

1. 谁选择下一位 Agent？
2. 谁判断整个协作结束？

选择下一位 Agent 可以由不同机制负责：

- Round Robin：代码维护数组下标，固定轮换；
- AI Selector：额外调用模型，根据能力和历史选择下一位；
- Plan-based：根据预先生成的计划和当前步骤选择 Agent。

因此：

> Orchestration 是持续的多 Agent 外层循环，不必然由 LLM 调度，也不必然高度自治。

同样，终止也不一定由 selector 模型决定。当前仓库把 termination 作为独立组件：

- 文本出现 `APPROVED`；
- 消息数达到上限；
- token、时间或函数调用达到条件；
- 用户取消；
- `max_iterations` 触发安全兜底。

“选谁”和“是否结束”是两项不同的系统责任。

### 5.3 状态形态

Round Robin 和 AI Selector 主要依赖 `shared_messages`：

~~~text
用户任务
  ↓
researcher response
  ↓
writer response
  ↓
reviewer response
  ↓
writer revision
  ↓
APPROVED
~~~

它的中心状态更像团队 transcript，而不是类型化的数据管道。

因此 Orchestration 通常是 **conversation-centric**：中心问题是基于当前协作历史，下一轮应该由谁采取什么行动。

### 5.4 理论优势和代价

优势：

- 运行时可以动态改变执行顺序；
- 适合反复批评、修订和协商；
- 新发现可以改变后续分工；
- Agent 能基于团队历史继续工作；
- 比静态图更容易处理路径事先未知的任务。

代价：

- selector 可能选错或重复选择；
- Agent 容易重复表达相同内容；
- transcript 会持续增长；
- 完成标准比终点节点更难定义；
- 运行路径不容易稳定复现；
- 自治选择本身也会产生模型成本；
- 如果没有清楚的状态和终止合同，容易出现无限讨论。

### 5.5 “讨论通过后进入实现”不是自动能力

下面是一种合理的上层组合：

~~~text
Discussion Orchestration
        ↓ APPROVED
Implementation Workflow
~~~

但它需要显式的阶段转换、handoff 或外层控制器。

Orchestrator 本身只负责持续协作，不会天然把“讨论完成”转换为“进入下一套实现流程”。当前仓库的 `HandoffOrchestrator` 也仍是占位实现。

## 6. Agent-as-Tool：一个 Agent 在调用另一个 Agent

### 6.1 核心模型

Agent-as-Tool 把一个完整 Agent Runtime 暴露为上层 Agent 可以调用的工具：

~~~text
Coordinator Agent
├── research_agent(task)
├── data_agent(task)
└── review_agent(task)
~~~

对子 Agent 而言，它仍然拥有自己的：

- instructions；
- model；
- tools；
- context；
- 内部 model-tool loop；
- usage 和 messages。

但对上层 Coordinator 来说，它被压缩为一个工具合同：

~~~text
输入：task
输出：ToolResult
~~~

### 6.2 完整调用链

~~~text
Coordinator 模型调用
  ↓
产生 Tool Call：research_agent(task="...")
  ↓
AgentAsTool.execute_stream()
  ↓
research_agent.run_stream()
  ↓
子 Agent 独立完成内部 Loop
  ↓
提取 last / last:N / all / custom result
  ↓
转换成 ToolResult
  ↓
转换成 Coordinator 的 ToolMessage
  ↓
Coordinator 模型继续
~~~

### 6.3 控制权和责任

上层 Coordinator 的模型决定：

- 是否调用某个专家；
- 给专家什么任务；
- 是否调用多个专家；
- 得到结果后如何综合；
- 是否需要再次调用；
- 最终如何回答用户。

专家 Agent 只拥有自己那次调用内部的控制权。

所以 Agent-as-Tool 是一种层级组合：

~~~text
上层 Agent：拥有最终任务
下层 Agent：拥有被委派的局部任务
~~~

### 6.4 信息隔离边界

当前 `AgentAsTool` 默认只向子 Agent 传递一个 task，并用子 Agent 自己的基础 context 副本执行：

~~~text
Coordinator context
      │
      │ task
      ▼
Specialist private run
      │
      │ extracted result
      ▼
Coordinator ToolMessage
~~~

结果可以选择：

- `last`：只返回最后一条消息；
- `last:N`：返回最后 N 条；
- `all`：返回完整消息；
- callable：使用自定义提取函数。

这不是单纯的输出格式选项，而是在定义子 Agent 与父 Agent 之间的信息边界。

流式执行中，子 Agent 的中间消息可以投影到 UI；但真正写进 Coordinator 模型上下文的主要是最终 ToolResult 对应的 ToolMessage。

因此必须区分：

> 用户界面能够观察到的子 Agent 过程，不等于父 Agent 模型下一轮能够读取的上下文。

### 6.5 理论优势和代价

优势：

- 能把专业能力封装为窄接口；
- 子 Agent 不必读取全部父上下文；
- 可以减少上下文污染和 token 消耗；
- 容易按专家划分工具和权限；
- 上层 Agent 保持统一的最终责任；
- 可以像普通工具一样组合或并发调用多个专家。

代价：

- task 描述不完整时，子 Agent 缺少必要背景；
- 过度压缩结果可能丢失证据和不确定性；
- 子 Agent 的内部失败可能被包装成普通工具错误；
- 如果父 Agent 无法追问或恢复子 Agent，会出现一次性黑盒调用；
- 嵌套模型调用会隐藏真实成本和延迟。

## 7. Agent-as-Tool 与可交互 Subagent 的边界

纯 Agent-as-Tool 更接近一次性函数：

~~~text
task → child agent run → result
~~~

而具有以下能力的系统已经超出简单 Agent-as-Tool：

- worker ID；
- 生命周期状态；
- `send_message`；
- `followup_task`；
- `wait`；
- 中途观察和继续调度；
- 长时间运行或后台执行。

这种模式更适合叫作 **Hierarchical Delegation Runtime**：

~~~text
一次性 Agent-as-Tool
        │
        │ 加入可寻址身份、状态和持续通信
        ▼
Hierarchical Subagent Delegation
        │
        │ 加入共享对话和轮流行动
        ▼
Team Orchestration
~~~

因此 Codex 当前的子代理机制包含 Agent-as-Tool 的思想，但由于主 Agent 可以观察状态、发送消息和继续派发任务，它比当前仓库的简单 `AgentAsTool` 更接近层级式子代理运行时。

判断标准是：

> 子 Agent 是调用后返回的函数，还是一个可以持续寻址和交流的 Actor / Worker？

## 8. 用同一个任务比较三种形态

任务：调研某项技术并写成一份经过审核的报告。

### Workflow

~~~text
research_agent
  → ResearchOutput
  → writer_agent
  → DraftOutput
  → reviewer_agent
  → ReviewOutput
~~~

开发者决定顺序；结果沿类型化边传递。

### Orchestration

~~~text
shared transcript
  → selector 选择 researcher
  → selector 选择 writer
  → selector 可能再次选择 researcher
  → selector 选择 reviewer
  → termination 判断是否结束
~~~

顺序可以在运行时变化；结果进入协作历史。

### Agent-as-Tool

~~~text
Coordinator
  → 判断需要调研
  → 调用 research_agent 工具
  → 获得研究结果
  → 自己继续撰写
  → 必要时调用 review_agent
  → 自己产生最终答案
~~~

Coordinator 始终拥有最终任务；子 Agent 只负责局部委派任务。

## 9. 最快的辨认方法

不要只看项目是否存在 `Coordinator`、`Planner` 或多个角色，而要看一个 Agent 完成后，结果进入哪里：

| 结果去向 | 更接近的模式 |
|---|---|
| 成为下一个节点的结构化输入 | Workflow |
| 追加到团队 shared transcript / shared state | Orchestration |
| 转换成上层 Agent 的 ToolMessage | Agent-as-Tool |

另一个判断维度是下一步的合法空间：

- Workflow：图预先规定合法后继；
- Orchestration：调度策略在候选 Agent 中选择；
- Agent-as-Tool：父 Agent 只能在已暴露的工具集合里委派。

## 10. 三种形态不是互斥分类

实际系统可以分层组合：

~~~text
外层确定性 Workflow
  │
  ├─ 数据准备 Step
  │
  ├─ 分析 Step
  │    └─ Coordinator Agent
  │         ├─ statistics_agent tool
  │         └─ chart_agent tool
  │
  ├─ 开放式审核 Orchestration
  │    ├─ writer
  │    └─ reviewer
  │
  └─ 人工审批 Step
~~~

因此这三个词更适合作为不同层级的控制原语，而不是给整个产品贴一个互斥标签。

一个系统可以同时包含：

- 外层 Workflow；
- 中间的多 Agent Orchestration；
- 单个 Agent 内部的 Agent-as-Tool；
- 最底层普通工具调用。

## 11. 从理论衍生出的设计原则

### 原则一：按不确定性分配自治权

不要因为“有多个 Agent”就选择自治 Orchestration。

- 路径已知的部分交给 Workflow；
- 局部专业任务交给 Agent-as-Tool；
- 只有下一步确实无法可靠预定义的部分，才使用开放式 Orchestration。

### 原则二：把确定性放在外层，把智能放在局部

生产系统通常更适合：

~~~text
可靠的外层 Workflow
  + 局部 Agent 判断
  + 窄接口专家工具
  + 少量开放式协作
~~~

这样既保留模型的适应能力，也能控制恢复、权限、成本和最终责任。

### 原则三：跨 Agent 边界必须有合同

合同不只包括输入输出 Schema，还包括：

- 子任务范围；
- 可见上下文；
- 可用工具和权限；
- 成功和失败表示；
- 证据和不确定性怎样返回；
- token、时间和重试预算；
- 是否允许副作用。

### 原则四：先确定最终责任人

多 Agent 系统最常见的问题不是没有能力，而是没有人对最终结果负责。

- Workflow：最终节点或结果聚合器负责；
- Orchestration：终止与 final synthesis 机制负责；
- Agent-as-Tool：上层 Coordinator 负责。

### 原则五：产物传递通常比纯聊天更可靠

当任务需要审计、恢复或多人协作时，应尽量让 Agent 交付明确产物：

- research brief；
- execution plan；
- code patch；
- review verdict；
- structured evidence；
- completion report。

纯 transcript 可以辅助协作，但不应总是承担全部状态和交接责任。

## 12. 当前仓库实现与理论模型的边界

本组件只保留会影响理论理解的实现提醒：

- `PicoAgentStep` 把 Agent 包装成 Workflow Step，清楚表达了“Agent 是节点”的概念；但当前异常包装方式不一定适合作为生产失败语义。
- `BaseOrchestrator` 清楚表达了选择、上下文、执行、状态更新和终止的外层循环；当前最终结果主要取最后一条 assistant message，尚未体现完整的 final synthesis 设计。
- `AgentAsTool` 清楚表达了 `task → child runtime → ToolResult`；当前交互和生命周期控制比完整 Subagent Runtime 简单。
- `HandoffOrchestrator` 尚未实现，因此“阶段放行”和显式 Agent 交接应作为理论衍生设计讨论，不能当作当前代码已具备的能力。

## 13. 关键源码入口

### Workflow

- `picoagents/src/picoagents/workflow/core/_workflow.py`
  - `BaseWorkflow.add_edge()`
  - `BaseWorkflow.chain()`
  - `BaseWorkflow.get_ready_steps()`
  - `BaseWorkflow.validate_workflow()`
- `picoagents/src/picoagents/workflow/core/_runner.py`
  - `WorkflowRunner.run_stream()`
  - `WorkflowRunner._execute_workflow_stream()`
- `picoagents/src/picoagents/workflow/steps/picoagent.py`
  - `PicoAgentStep.execute()`

### Orchestration

- `picoagents/src/picoagents/orchestration/_base.py`
  - `BaseOrchestrator.run_stream()`
  - `select_next_agent()`
  - `prepare_context_for_agent()`
  - `update_shared_state()`
- `picoagents/src/picoagents/orchestration/_round_robin.py`
- `picoagents/src/picoagents/orchestration/_ai.py`
- `picoagents/src/picoagents/orchestration/_plan.py`
- `picoagents/src/picoagents/orchestration/_handoff.py`

### Agent-as-Tool

- `picoagents/src/picoagents/agents/_agent_as_tool.py`
  - `AgentAsTool.execute()`
  - `AgentAsTool.execute_stream()`
  - `AgentAsTool._extract_result()`
- `picoagents/src/picoagents/agents/_base.py`
  - `BaseAgent.as_tool()`
- `picoagents/src/picoagents/agents/_agent.py`
  - `Agent._execute_tool_call()`

## 14. 一句话总结

> Workflow 让图调用 Agent；Orchestration 让调度器持续运行一组 Agent；Agent-as-Tool 让一个 Agent 把另一个 Agent 当成具有独立内部 Loop 的专业工具。真正的架构选择取决于控制权、状态和结果边界，而不是 Agent 数量。
