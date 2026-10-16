# 组件 06：Durable Human-in-the-loop

> 状态：已完成第一轮提炼，待共同深入
>
> 核心判断：人工审批不是在 Agent Loop 中插入一个 `input()`，而是把一次运行暂停成可持久化状态，并在另一个时间、进程甚至设备上安全恢复。

## 1. 基础审批流程隐藏了什么

表面流程是：

~~~text
模型生成 tool call
       ↓
工具需要审批
       ↓
用户 approve / reject
       ↓
继续执行
~~~

如果审批立即发生在同一个进程里，这很简单。但真实产品中：

- 用户可能几小时后才操作；
- 服务实例可能已经重启；
- 审批链接可能被点击多次；
- 同一个请求可能被多个审批人同时处理；
- tool 权限或参数校验规则可能已经变化。

因此审批必须是一种 durable protocol。

## 2. 审批对象必须绑定原始意图

一个审批请求至少应保存：

~~~text
approval_request_id
workflow / session / run ID
tool_call_id
tool name and version
完整参数或不可变参数快照
参数摘要和用户可读说明
requester agent identity
创建时间、过期时间
required approver policy
context / policy version
current status
~~~

最重要的是：审批的是一份已经固定的 tool call，而不是一个模糊意图。

例如用户批准的是：

~~~text
send_email(to=A, attachment=v3.pdf)
~~~

恢复后不能重新调用模型生成参数，否则它可能变成：

~~~text
send_email(to=B, attachment=v4.pdf)
~~~

然后错误地沿用原审批结果。

## 3. 正确的暂停边界

暂停前应 durable commit：

1. 原始 tool call；
2. approval request；
3. 当前 execution position；
4. 必要 artifact 和上下文版本；
5. “工具尚未执行”的状态。

然后运行进入：

~~~text
WAITING_FOR_APPROVAL
~~~

恢复后直接处理已经固定的 tool call：

- approved：执行该调用；
- rejected：生成确定性的拒绝 ToolResult，再让 Agent 决定下一步；
- expired：按策略拒绝、重新申请或交给人工恢复；
- policy changed：使旧审批失效，重新验证。

## 4. 审批投递和工具执行都可能重复

真实网络通常是 at-least-once delivery：

~~~text
用户点击批准
  ↓
服务已处理，但响应丢失
  ↓
前端重试提交批准
~~~

因此审批状态迁移需要 compare-and-set：

~~~text
PENDING → APPROVED
PENDING → REJECTED
~~~

只有第一次合法迁移成功。之后相同请求只能返回已有结果，不能再次触发工具。

工具执行同样必须使用稳定 idempotency key：

~~~text
idempotency_key = approval_request_id / tool_call_id
~~~

## 5. 审批不是授权全部后续行为

一次批准的 scope 应尽可能窄：

- 指定工具；
- 指定参数；
- 指定资源范围；
- 指定时间窗口；
- 指定最大数量或金额；
- 指定调用次数；
- 指定执行主体。

如果工具参数发生实质变化，必须产生新的 approval request。

对于批量动作，还要明确用户批准的是：

- 整个 batch；
- 每一项独立操作；
- 一个带上限的 policy grant。

## 6. Reject 也是正式执行结果

拒绝不应简单抛异常或丢弃 tool call。它应生成明确的 ToolResult：

~~~text
tool_call_id: ...
status: rejected
reason: ...
approved_by: ...
timestamp: ...
~~~

这样 Agent 才能基于拒绝事实：

- 提出无副作用方案；
- 修改计划；
- 请求更窄权限；
- 向用户解释无法继续的部分；
- 正常结束运行。

## 7. 与 Checkpoint 的关系

Durable approval 是 checkpoint 的一种特殊情况，但风险更高：

~~~text
普通 checkpoint：恢复计算进度
approval checkpoint：恢复一项带权限和副作用约束的未决事务
~~~

它必须同时满足：

- execution 可恢复；
- 审批决定不可被篡改；
- tool call 参数不可漂移；
- 审批人身份可审计；
- 重复恢复不会重复产生副作用。

## 8. 仓库实现提供的证据

`AgentContext` 当前保存：

- `pending_approval_requests`；
- `approval_responses`；
- `pending_tool_calls`；
- approval request 与原始 tool call 的绑定。

WebUI 在后续请求中把 approval response 注入原 session context，再继续 `agent.run_stream()`。

它已经表达了两个正确边界：

1. 审批必须引用原始 `tool_call_id`；
2. 审批后可以不追加新的用户消息，直接恢复原 context。

但整体仍偏向进程内 / session 内状态，没有完整展示：

- approval record 的持久化；
- 多实例并发处理；
- 过期与 policy version；
- compare-and-set 状态迁移；
- tool execution 的幂等提交。

## 9. 生产设计不变量

1. 审批永远绑定不可变的原始 tool call；
2. 参数实质变化必须重新审批；
3. approval request 状态只能发生一次终局迁移；
4. 重复审批请求返回已有结果，不重复执行工具；
5. 审批后续跑时不重新让模型生成已批准参数；
6. approval、execution 和 side-effect 拥有可关联的稳定 ID；
7. 拒绝、过期、撤销和 policy change 都有明确状态；
8. 审批日志不把敏感参数无控制地复制到 telemetry。

## 10. 关键源码入口

- `picoagents/src/picoagents/context.py`
  - `ToolApprovalRequest`
  - `ToolApprovalResponse`
  - `AgentContext.add_approval_request()`
  - `AgentContext.add_approval_response()`
- `picoagents/src/picoagents/agents/_agent.py`
  - tool approval handling
- `picoagents/src/picoagents/webui/_execution.py`
  - approval response injection and session resume
- `examples/tools/approval_example.py`

## 11. 一句话总结

> 真正的 Human-in-the-loop 是一套可跨进程恢复的权限事务：冻结待批准调用、持久化决定、保证重复投递安全，并在批准后继续原执行位置，而不是重新向模型询问一次。
