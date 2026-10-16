# 组件 04：Checkpoint、重放、幂等与副作用

> 状态：已完成第一轮提炼，待共同深入
>
> 核心判断：Checkpoint 不是简单的“保存进度”，而是 durable execution 的一部分。它必须和重放语义、节点幂等性及外部副作用协议一起设计。

## 1. Checkpoint 能解决什么

一个基础 checkpoint 可以保存：

- workflow identity；
- workflow structure hash；
- execution state；
- 每个 step 的状态、输入、输出和错误；
- completed / pending step IDs；
- checkpoint 时间和类型。

恢复时，调度器读取已完成集合，只重新调度仍未完成的节点。

它能减少重复计算，但不能独自保证外部行为只发生一次。

## 2. 无法被 checkpoint 消除的 crash window

~~~text
节点调用支付、邮件或 GitHub API
             ↓
外部系统已经成功执行
             ↓
当前进程在写 checkpoint 前崩溃
             ↓
恢复时仍认为节点没有完成
             ↓
节点再次执行外部调用
~~~

这是无法仅靠“每个 step 后保存 checkpoint”彻底消除的窗口。

除非 workflow state 和外部系统处在同一个事务中，否则 Runtime 通常只能提供：

~~~text
at-least-once execution
        +
幂等节点 / 去重协议
~~~

而不是天然的 exactly-once execution。

## 3. 幂等不等于简单重试

一个节点需要稳定的操作 identity：

~~~text
operation_id = workflow_run_id + node_id + logical_attempt
~~~

外部工具调用时携带 idempotency key：

~~~text
send_email(
  idempotency_key="run-123/node-send-report",
  ...
)
~~~

如果恢复后再次收到相同 key，工具层应返回第一次调用的结果，而不是再次产生副作用。

真实设计还应保存：

- tool call ID；
- 参数摘要或 hash；
- side-effect status；
- 外部系统返回的 resource ID；
- 首次提交时间；
- 是否允许人工补偿。

## 4. Retry、Replay 和 Compensation 的区别

### Retry

同一次逻辑操作失败后再次尝试，期望得到同一种效果。

### Replay

根据事件或 checkpoint 重建执行状态。Replay 可能再次经过原节点，因此要求节点逻辑可重入。

### Compensation

副作用已经成功，但后续流程失败，通过另一项业务操作抵消它，例如退款、撤销发布或发送更正邮件。

Compensation 不是数据库 rollback。很多真实副作用无法恢复成“从未发生过”。

## 5. Checkpoint 兼容性不只是图结构 hash

仓库会计算 workflow structure hash，内容包括：

- Step ID 和类型；
- Edge 连接与 condition 类型；
- Start / End step IDs。

结构变化时拒绝恢复，这个方向正确。但真实 Agent Workflow 还需要绑定：

~~~text
graph version
node implementation version
prompt / instructions version
model and decoding config
tool schema version
artifact schema version
policy / permission version
~~~

因为图没有变化不代表程序语义没有变化。

例如节点 ID 仍叫 `review`，但 prompt 已从“检查事实错误”改成“润色语气”，旧 checkpoint 恢复到新代码上就可能产生语义断裂。

## 6. In-flight 节点怎样恢复

崩溃时节点可能处于：

- 尚未开始；
- 正在模型调用；
- 模型已返回、工具未调用；
- 工具已调用、结果未持久化；
- 结果已持久化、节点状态未更新。

因此只保存 `PENDING / RUNNING / COMPLETED` 不足以覆盖高风险工具。可以把一个 Agent 节点拆为事件：

~~~text
node_started
model_decision_recorded
tool_call_prepared
tool_call_committed
tool_result_recorded
artifact_committed
node_completed
~~~

恢复逻辑根据最后一个 durable event 决定：

- 安全重算；
- 查询外部操作结果；
- 继续后半段；
- 请求人工确认；
- 执行补偿。

## 7. Artifact 应与 transcript 分开

恢复节点时最可靠的边界不是重新播放全部自然语言历史，而是读取已提交 artifact：

~~~text
artifact_id
schema_version
producer_node
content_hash
payload
evidence
~~~

Transcript 可以帮助审计，但下游依赖应尽可能绑定稳定 artifact，而不是依赖重新解释历史消息。

## 8. 仓库实现提供的证据

当前仓库：

- checkpoint 保存完整 `WorkflowExecution`；
- 记录 completed 和 pending steps；
- 支持自动按完成 step 数保存；
- 恢复前校验 workflow structure hash；
- structure mismatch 时拒绝恢复。

它表达了“恢复必须验证程序兼容性”的正确方向，但没有完整解决：

- 外部副作用去重；
- in-flight tool call 的精细状态；
- prompt / model / tool 版本兼容；
- event log 与物化 execution state 的关系。

## 9. 生产设计不变量

1. 每个可能产生副作用的节点都拥有稳定 operation ID；
2. 重试次数和 logical operation identity 分开；
3. checkpoint 写入失败不能让已经执行的副作用失去身份；
4. 恢复前验证完整 execution contract 版本；
5. 下游消费 durable artifact，而不是临时内存对象；
6. 无法幂等的操作必须有人工恢复或 compensation 策略；
7. 不宣称 exactly-once，除非能说明跨系统事务怎样成立。

## 10. 关键源码入口

- `picoagents/src/picoagents/workflow/core/_checkpoint.py`
  - `WorkflowCheckpoint`
  - `CheckpointStore`
  - `CheckpointConfig`
- `picoagents/src/picoagents/workflow/core/_workflow.py`
  - `compute_structure_hash()`
- `picoagents/src/picoagents/workflow/core/_runner.py`
  - checkpoint resume path
  - `validate_checkpoint()`

## 11. 一句话总结

> Checkpoint 只能告诉 Runtime 从哪里继续；要避免恢复后重复发邮件、提交代码或扣款，必须额外建立稳定操作身份、幂等工具和副作用提交协议。
