# 组件 05：Agent 关键路径、工具等待与状态驻留

> 状态：待共同阅读

## 1. 学习目标与边界

理解多轮 Agent 任务中的依赖、并行、工具等待和状态寿命，重点阅读第 3 章 3.2。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
模型调用
  ↓
工具调用 / 分支
  ↓
等待依赖汇合
  ↓
状态写回 Context
  ↓
下一轮模型调用
```

## 4. 关键输入、输出和状态

重点记录模型时间、工具时间、依赖关系、并行分支、Context 增长和等待期间的资源驻留。

## 5. 公式与数量级

共同阅读时区分工作量总和与依赖图中的 Critical Path。

## 6. 正文、代码与实验入口

- `manuscripts/03-推理与训练负载.md`：3.2；
- `calculations/src/infra_calc/topics/agent_trace.py`；
- `calculations/src/infra_calc/topics/execution_dag.py`；
- `calculations/src/infra_calc/topics/request_dag.py`；
- `calculations/src/infra_calc/topics/strategy_record_cost.py`；
- `experiments/ch03/03-03/README.md`；
- `experiments/ch03/03-04/README.md`。

## 7. 与 Agent Harness 的连接

直接连接 Agent Loop、并行 Tool Call、Subagent、Thinking、Context 增长和恢复策略。

## 8. 共同阅读时要回答的问题

- 为什么所有步骤耗时相加不一定等于任务延迟？
- 工具等待时模型状态和环境状态是否仍占资源？
- 并行分支什么时候真的缩短关键路径？
- Thinking 关闭或加速某一轮为什么不一定改善最终成功时间？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

本组件先研究负载和依赖，不提前讨论完整调度器实现。

