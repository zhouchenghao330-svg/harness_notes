---
created: 2026-10-31
tags:
  - 开发经验
  - Temporal
  - Workflow
  - 故障恢复
  - Agent
---

# Temporal恢复设计思想

> 可以把 Temporal 抽象成一套**可靠执行系统的设计方法**：把流程决策、外部操作、已确认的执行事实、调度与故障恢复分开，再用持久化的事件历史把它们接起来。这里讨论可借鉴的思想，不是 AI-Human 已决定引入 Temporal。

记录时间：2026-10-31。依据当天对 [Temporal 官方架构与执行文档](https://docs.temporal.io/workflow-execution) 的阅读整理；“对 AI-Human 的启发”是项目设计判断，不是 Temporal 官方对本项目的建议。

## 1. 先看整体分工

```text
Workflow 定义（业务流程代码）
  决定步骤、顺序、条件、等待与分支
                 ↓ 由 Workflow Worker 运行并发出命令
Temporal Service
  持久化 Event History，调度 Workflow/Activity Task，管理定时器与重试
                 ↓ 派发任务
Worker
  运行 Workflow 代码；Activity Worker 执行数据库、模型、外部 API 等操作
                 ↓ 返回结果或失败
Event History
  记录服务端已确认的关键事件，供后续重放和恢复
```

图中的 Event History 是 Service 持久化的执行记录，不是另一个运行业务代码的角色。Temporal Service 不执行用户的 Workflow 或 Activity 代码；Worker 从任务队列接活、运行代码并回报。等待审批或定时器时，不需要某个 Worker 始终占着这次 Workflow 的内存；后续可以由同一或另一个 Worker 根据历史继续。[Worker 与 Service 的职责](https://docs.temporal.io/workers)、[持久化定时器](https://docs.temporal.io/workflow-execution/timers-delays)

## 2. 核心机制拆成五件事

| 机制 | 负责什么 | 恢复时的意义 |
| --- | --- | --- |
| Workflow | 用代码表达流程顺序、判断、循环、等待和下一步决策；不必是一张静态 DAG | 重放时按相同输入和历史产生相匹配的命令 |
| Activity | 把数据库、LLM、外部 API、发消息等操作放到流程决策之外 | 操作可设置超时和重试；完成结果进入历史，重放时复用 |
| Event History | 记录 Workflow 生命周期中已确认的关键事件，如任务、Activity 结果、定时器及消息 | 提供恢复所依据的事实，而不是逐行内存快照 |
| Replay | Worker 重新运行 Workflow 代码，用历史匹配先前命令并恢复状态 | 已在历史中完成的 Activity 不因 Workflow 重放而重新执行 |
| 调度与恢复 | Service 持久化状态并安排任务、定时器、超时及重试 | Worker 中断后仍可继续推进长时间流程 |

确定性约束针对 **Workflow 决策路径**：相同历史下，重放应产生与历史匹配的 Workflow 命令。非确定性的数据库查询、外部请求和 LLM 调用应放在 Activity 等受管理的边界，而不是在 Workflow 代码重放时直接再次调用。正在运行的 Workflow 还要考虑代码版本变化，否则新代码可能生成与旧历史不匹配的命令。[Workflow 的确定性与版本变化](https://docs.temporal.io/workflow-definition)、[Replay 与命令](https://docs.temporal.io/workflow-execution#replay)

## 3. Replay 与“记住当前步骤”有什么不同

假设流程是：

```text
开始 → API A 完成 → API B 完成 → API C 执行期间 Worker 中断
```

仅保存 `current_step=C`，恢复时还要自己回答：A、B 的结果在哪里？C 是否已经产生副作用？这次调用能否重试？Temporal 的思路是保存已确认的事件。Worker 再运行 Workflow 代码时，A、B 的已记录结果由历史提供；运行到尚未有完成事件的地方，系统再根据任务与重试状态推进。它恢复的是**由历史重建的流程状态**，不只是跳到某个代码行号。[Event History 的恢复作用](https://docs.temporal.io/encyclopedia/event-history)

这也要分清两个“再次执行”：**Workflow 代码会重放，已记录完成的 Activity 不随重放再执行；未被确认完成的 Activity 可能按重试策略再次执行。**如果 C 已向外部系统写入成功，但 Worker 在报告结果前崩溃，历史没有完成事实，C 仍可能被重试。外部副作用需要稳定的幂等键、目标侧去重或先查实际状态。Temporal 保证执行记录和调度，并不自动替外部系统实现“副作用恰好一次”。[Activity 幂等与未报告结果窗口](https://docs.temporal.io/activity-definition#idempotency)

## 4. 与 Checkpoint 的比较：两种恢复主线

下表比较的是**以状态快照为主的简化方案**与 **Temporal 的事件历史加重放机制**，不是断言所有 Checkpointer 只存快照，也不是说两者不能组合。例如 LangGraph 的 Checkpointer 还可保存阶段内的 Pending Writes。

| 维度 | 以 Checkpoint 为主 | Event History + Replay |
| --- | --- | --- |
| 主要保存 | 某个阶段的可恢复状态，以及必要的任务结果 | 已确认的流程事件与操作结果 |
| 主要恢复方式 | 加载状态，由 Runtime 决定从哪里继续 | 重放 Workflow 决策，依据事件历史重建进度 |
| 代码约束 | 状态可序列化，恢复协议能识别已完成工作 | Workflow 命令序列还须可确定性重放，并处理代码版本 |
| 需要自行处理 | 快照与任务结果的一致性、遗漏的执行窗口 | 历史增长、重放成本、版本兼容；外部操作仍须幂等 |

动态 Agent Loop 的分支会随模型输出和工具结果变化，直接采用 Checkpoint 加明确的工具回执可能更自然；长期等待、跨服务且要求流程连续的业务，事件历史与调度机制更值得研究。选哪种方式应先看任务的失败窗口、恢复粒度和副作用，不能只按技术名称判断。详见 [[开发经验/LangGraph Checkpointer 实现以及探讨|LangGraph Checkpointer 的状态、Writes 与恢复]]。

## 5. 对 AI-Human 值得吸收的三条原则

### 执行事实与对话历史分离

Message History 面向模型，用来提供对话语境；执行记录面向 Runtime，用来回答“哪次操作已经被系统确认完成”。同一条工具回执可以投影给模型，但不能仅靠模型可见的消息文本推断外部操作是否真的生效。客户端 SSE 事件重放用于恢复展示，也不自动等于服务端任务恢复。

### 给操作定义清楚的完成边界

至少区分：尚未派发、正在执行、已确认完成、确定失败、结果未知。对于“请求已发出但没有可靠回执”的写入，直接重试可能产生重复副作用。Runtime 应保存操作身份和必要事实，并按工具契约决定重试、查证、恢复或交给人处理。

### 让执行策略与业务语义各司其职

只读 SQL 查询可以考虑有界重试，但仍要限制成本、超时和可见范围；创建工单、发送通知等写入操作，可能先要用幂等键或查询目标状态确认上次是否已经生效。通用 Runtime 负责可靠调度和状态边界，业务工具说明副作用、幂等方式与成功条件。不能把所有失败简单交给 Agent 再猜一次。相关原则见 [[开发经验/Harness 设计(重试,幂等...)|Harness 设计：重试、幂等与安全写入]]。

## 6. 当前取舍

现阶段值得借鉴的是：**在必要的执行边界记录“系统已确认发生了什么”，再据此决定下一步允许做什么。**这不要求 AI-Human 立刻采用 Temporal 的 SDK、部署架构或完整事件重放。主 Agent Loop、SQL Workflow、工具回执和状态持久化各有现成的设计边界；应先核对真实故障场景，再决定是否需要更细的执行事件、快照或两者组合。

如果未来要采用完整事件重放，还须评估 Workflow 的确定性与版本兼容、历史增长、重放成本、等待/信号机制，以及外部副作用的幂等设计。当前这篇笔记只记录设计思想，不作为迁移计划或已完成的 AI-Human 恢复能力说明。

## 官方资料

- [Workflow Execution：Replay、命令与恢复](https://docs.temporal.io/workflow-execution)
- [Event History：事件如何支持状态重建](https://docs.temporal.io/encyclopedia/event-history)
- [Workers：Worker 与 Temporal Service 的职责](https://docs.temporal.io/workers)
- [Workflow Definition：确定性与代码变化](https://docs.temporal.io/workflow-definition)
- [Activity Definition：幂等和结果未记录的窗口](https://docs.temporal.io/activity-definition#idempotency)
- [Timers and Start Delays：持久化等待](https://docs.temporal.io/workflow-execution/timers-delays)
