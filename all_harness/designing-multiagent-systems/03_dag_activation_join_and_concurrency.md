# 组件 03：DAG 的激活、汇合与并发确定性

> 状态：已完成第一轮提炼，待共同深入
>
> 核心判断：DAG 不是画出来的流程图，而是一套关于节点何时被激活、何时可以运行、读取哪些产物以及怎样结束的执行语义。

## 1. 最小模型

~~~text
          ┌── B ──┐
A ────────┤       ├── D
          └── C ──┘
~~~

这张图至少包含四个互相独立的问题：

1. A 完成后，B 和 C 是否都被激活？
2. D 等全部被激活的前驱，还是任意一个前驱？
3. D 怎样取得并合并 B、C 的输出？
4. B、C 并发完成顺序不同时，结果是否一致？

只定义节点和边，不能回答这些问题。

## 2. Topology fan-out 不等于 Runtime activation

如果 A 有两条条件边：

~~~text
A -- type=code --> B
A -- type=docs --> C
~~~

图的出度是 2，但一次运行中可能只激活一条边。

因此必须区分：

- **Topology fan-out**：静态图中存在多少个后继；
- **Activated fan-out**：本次 execution 中实际产生了多少个可运行后继。

节点的 `PENDING` 也有歧义：

- 已激活，正在等待依赖；
- 尚未判断是否激活；
- 条件不成立，本次永远不会运行。

第三种状态不能永远留在 `PENDING`，否则后续 join 可能永久等待。它需要被标记为 `SKIPPED`，或者通过 execution token 明确表示“不属于本次激活子图”。

仓库的 `StepStatus` 定义了 `SKIPPED`，但调度主循环没有形成完整的未激活分支关闭协议。这说明枚举中出现 `SKIPPED` 并不等于条件 DAG 已经拥有完整语义。

## 3. Control fan-in 不等于 Data fan-in

D 的汇合包含两个正交问题。

### 3.1 Control fan-in

决定 D 什么时候 ready：

- AND join：所有激活前驱都成功后运行；
- OR join：任意一个符合条件的前驱完成后运行；
- N-of-M join：M 个候选中达到 N 个结果后运行；
- Quorum join：按权重或置信阈值判断；
- Race join：采用第一个成功结果，并取消其余分支。

### 3.2 Data fan-in

决定 D 实际读取什么：

~~~python
D.input = reduce({
    "research": B.output,
    "analysis": C.output,
})
~~~

仓库的 `get_ready_steps()` 对无条件入边实现了类似 AND join：所有依赖完成后节点才 ready。

但 `_prepare_step_input()` 会扫描依赖，取得第一个完成依赖的 output 后直接停止。这会产生明显的不一致：

~~~text
控制平面：确认 B、C 都完成
数据平面：只把其中一个结果交给 D
~~~

这暴露出一条重要设计原则：

> Edge 首先表达执行依赖；多上游数据汇合必须拥有单独、显式、类型化的 reducer 合同。

## 4. 条件分支重新汇合

假设只有 B 被激活，而 C 被跳过：

~~~text
          ┌── B(active) ──┐
A ────────┤               ├── D
          └── C(skipped) ─┘
~~~

此时“等待全部前驱”有两种完全不同的解释：

1. 等图上的所有前驱：D 永远无法启动；
2. 等本次被激活的所有前驱：B 完成后 D 可以启动。

生产系统通常应该以 **activated predecessor set** 为准：

~~~text
ready(D) = all(parent terminal for parent in activated_predecessors(D))
~~~

其中 terminal 不一定只包含 `COMPLETED`，还要由 join policy 决定是否接受：

- `SKIPPED`；
- `FAILED`；
- `CANCELLED`；
- 超时后产生的降级结果。

## 5. 并发完成顺序与确定性

仓库调度器会并发启动 ready steps，并使用 `FIRST_COMPLETED` 消费先结束的任务。这意味着节点完成顺序天然不确定。

如果 B、C 直接修改同一个共享字典：

~~~text
B: state["summary"] = result_b
C: state["summary"] = result_c
~~~

结果就是不可预测的 last-write-wins。

更可靠的设计是：

~~~text
B → artifacts.B
C → artifacts.C
        ↓
deterministic reducer
        ↓
state.summary
~~~

Reducer 应至少满足下面一项：

- 输入先按稳定 key 排序；
- 操作满足交换律与结合律；
- 明确优先级和冲突规则；
- 产生冲突对象，交给后续节点裁决。

## 6. Agent 节点的完成不能只看“返回了一段话”

Agent DAG 中，一个节点完成通常应交付：

- 结构化 artifact；
- completion status；
- 使用过的证据或引用；
- 剩余不确定性；
- 是否产生外部副作用；
- 可供重试和审计的 metadata。

LLM 返回最终消息，只说明一次模型交互结束，不自动等于业务节点完成。

## 7. DAG 与开放循环的边界

DAG 不允许 cycle，因此不适合直接表达无限讨论或反复重规划。

迭代通常有三种放置方式：

1. 封装在单个节点内部，DAG 只观察节点最终状态；
2. 使用有限展开，例如最多三轮 review-revise；
3. 将外层提升为状态机或动态图，DAG 只负责一次确定性 execution。

DAG 的主要价值不是“更智能”，而是：

- 显式依赖；
- 并行调度；
- 有界执行；
- 清晰恢复边界；
- 可审计的节点产物。

## 8. 生产设计不变量

一个可用的 DAG Runtime 应明确保证：

1. 每个节点在本次运行中最终进入 terminal state；
2. readiness 基于 activated predecessors，而不是只看静态入边；
3. control join 和 data reducer 分开定义；
4. 并发节点不直接无序覆盖同一状态字段；
5. 节点产物拥有稳定 schema 和唯一 identity；
6. 重试不会破坏已经产生的外部副作用；
7. workflow 的完成条件与分支激活语义一致。

## 9. 关键源码入口

- `picoagents/src/picoagents/workflow/core/_models.py`
  - `StepStatus`
  - `EdgeCondition`
- `picoagents/src/picoagents/workflow/core/_workflow.py`
  - `get_ready_steps()`
  - `_evaluate_edge_condition()`
- `picoagents/src/picoagents/workflow/core/_runner.py`
  - `_execute_workflow_stream()`
  - `_prepare_step_input()`
- `examples/workflows/conditional.py`

## 10. 一句话总结

> DAG 真正管理的是本次 execution 的激活子图和偏序关系；“所有上游都完成”只解决控制汇合，不能替代数据合并、分支关闭和并发确定性设计。
