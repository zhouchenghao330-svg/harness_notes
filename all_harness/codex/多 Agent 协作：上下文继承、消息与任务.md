---
created: 2026-10-28
tags:
  - 开发经验
  - Agent
  - 多智能体
  - 上下文工程
---

# 多 Agent 协作：上下文继承、消息与任务

> 创建子 Agent 时，需要决定它带着多少父上下文开始；协作过程中，还需要区分“传一条消息”和“让对方继续执行任务”。

> 状态：前期资料记录。Codex 尚未开始系统拆解，本篇不表示已共同阅读或理解。

这篇从相关讨论出发，并核对本地 Codex 仓库的 Multi-Agent V2 实现。源码版本：`c0e88f3c04219aaa032aaf4b20a8c3284cc81449`。具体行为以对应版本、实际启用的工具和运行时为准；Codex Room 中独立成员的通信规则不能直接等同于这里的原生子 Agent 工具。

## 1. 创建子 Agent：决定带多少上下文

假设主 Agent 分析一个系统，分别让三个子 Agent 检查 SQL 校验、错误处理和上下文压缩。它们各自需要的信息不同，未必都需要主 Agent 已经积累的全部讨论。

当前源码中，Multi-Agent V2 的 `spawn_agent` 工具描述给出两种上下文继承方式：

| `fork_turns` | 含义 | 适用考虑 |
| --- | --- | --- |
| `all`（默认） | 继承父 Agent 的历史上下文 | 子任务依赖前面大量讨论或复杂约束 |
| `none` | 不复制父 Agent 对话历史 | 任务边界清晰，可在启动消息中提供必要背景 |

`none` 不等于子 Agent 没有任何上下文或运行配置：它仍有自己的启动任务、环境和适用配置；这里说的是**不复制父对话历史**。如果任务说明不完整，子 Agent 可能不知道项目背景、用户限制或已经做过的决定。

当历史很长、子任务很独立时，使用较少的继承上下文可能减少重复输入。但总成本还要看缓存、服务计费、子 Agent 后续查阅以及重复沟通，不能直接将父历史长度乘以子 Agent 数当成实际费用。

一个实用的启动说明可以包含：任务目标、需要查看的路径、必要背景、限制条件、期望返回的结果，以及是否允许修改文件。

### 一个版本细节：数字不表示最近 N 轮

文章提到 `3` 表示最近三轮，但本次核对的 V2 工具说明只支持 `all` 和 `none`。更值得注意的是，当前内部解析器为了兼容旧值，接受正整数形式的字符串，却将其作为 **FullHistory** 处理，并不会限制为最近 N 轮。因此不能把 `fork_turns="3"` 当成只继承三轮的方法；应遵循当前公开工具说明。

## 2. 消息和后续任务是不同的操作

| 工具 | 做什么 | 对空闲目标的行为 |
| --- | --- | --- |
| `send_message` | 给已有 Agent 传递一条消息 | 不主动触发新的 Turn |
| `followup_task` | 给已有非 root Agent 下发后续任务 | 如果目标空闲，触发新的 Turn |

比如，A 已经在检查 SQL 校验，主 Agent 得知一个新接口约束，可以用 `send_message` 将信息提供给 A；如果 A 已经做完并空闲，主 Agent 要求它按新接口重新检查，则可以用 `followup_task`。

目标正在运行时，后续任务会在模型消息边界，或当前工具调用完成后交付；不是强制打断正在执行的工具。`followup_task` 不能以 root Agent 为目标。具体何时被处理还要看运行状态，工具调用成功本身不代表任务已经完成。

源码中，两者共用消息提交路径，但分别使用 `QueueOnly` 与 `TriggerTurn`。普通消息与新任务也以不同的消息类型进入 Agent 输入。**发送内容是通信，是否触发执行是调度行为**；两者可以共享基础设施，又保持不同语义。

## 3. Mailbox 及值得检查的边界

Mailbox 可以理解成 Agent 的待处理消息位置：发送方不必一直等接收方执行完，接收方在合适的处理时机取出内容。当前代码还会在本地会话被卸载时保留待处理的普通消息，并有测试覆盖消息重新取出与特定场景的顺序。

设计多 Agent Runtime 时，可以区分几个概念：

```text
消息被接受或投递 → 接收方取出 / 消费 → 根据内容行动 → 任务完成
```

这些是分析协作状态的概念划分，不表示当前实现已经对每一层都提供同名状态或端到端保证。尤其“消息消费”不等于模型已经正确理解，也不等于工作完成。

进一步设计或评估时，可以检查：忙碌 Agent 在何时看到消息、重复投递如何处理、并发发送时有哪些顺序保证、Agent 卸载或终止后消息如何处理，以及结果如何返回。测试覆盖一个具体场景，不等于已经证明所有故障条件下都不会丢消息或重复执行。

## 4. 源码锚点

本地仓库：`/Users/zhouchenghao/python_file/codex`，版本 `c0e88f3c04219aaa032aaf4b20a8c3284cc81449`。

| 内容 | 源码 |
| --- | --- |
| 公开工具描述、继承参数和消息行为 | `codex-rs/core/src/tools/handlers/multi_agents_spec.rs`，`spawn_agent_common_properties_v2()`、`create_send_message_tool()`、`create_followup_task_tool()` |
| `all` / `none` 与旧数字的实际解析 | `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs`，`SpawnAgentArgs::fork_mode()` |
| 两种消息模式及共同提交路径 | `codex-rs/core/src/tools/handlers/multi_agents_v2/send_message.rs`、`followup_task.rs`、`message_tool.rs` |
| 模式如何转为消息或新任务 | `codex-rs/core/src/agent/control/delivery.rs` |
| root 目标限制与 Agent 投递 | `codex-rs/core/src/agent/control/api.rs` |
| 待处理消息与输入队列 | `codex-rs/core/src/agent/control/mailbox.rs`、`codex-rs/core/src/session/input_queue.rs` |
| 本地卸载、恢复与顺序的具体测试 | `codex-rs/core/tests/suite/agent_mailbox.rs`，`idle_mail_survives_eviction_and_preserves_followup_order()` |

相关但不同的工程技巧：[[all_harness/codex/arg0：同一个可执行文件承担多个命令|arg0：同一个可执行文件承担多个命令]]。
