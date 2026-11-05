# 01 Agent Loop And Turn Runtime

## 这个组件研究什么

agent 每一轮是怎么跑起来的。

## 应该回答的问题

- 一轮 turn 的输入是什么
- 主循环怎么组织
- 模型调用与工具调用怎么交错
- 中断、重试、预算、迭代上限怎么处理
- 一轮结束时哪些后处理会发生

## 典型关注点

- `run_conversation`
- turn lifecycle
- interrupt / retry / resume
- iteration budget
- post-turn hooks
