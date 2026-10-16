# Designing Multi-Agent Systems 组件索引

> 本索引已从八个教学主题收缩为四个工程主题。“已初步提炼”表示已经完成全仓库筛选和第一轮结论写入，后续只在共同讨论后继续补充。

## 1. 已保留的背景

| 编号 | 主题 | 定位 | 状态 |
|---|---|---|---|
| 02 | [[02_multiagent_forms\|Workflow / Orchestration / Agent-as-Tool]] | 三种控制权边界，作为后续讨论的共同背景 | 已共同阅读 |

## 2. 高增益主题

| 编号 | 主题 | 主要源码入口 | 真正要研究的问题 | 状态 |
|---|---|---|---|---|
| 03 | [[03_dag_activation_join_and_concurrency\|DAG：激活、汇合与并发]] | `workflow/core/_workflow.py`、`_runner.py`、`_models.py` | 激活集合、control/data fan-in、未选分支、并发确定性 | 已初步提炼 |
| 04 | [[04_checkpoint_replay_and_idempotency\|Checkpoint、重放与幂等]] | `workflow/core/_checkpoint.py`、`_runner.py` | crash window、at-least-once、副作用去重、版本兼容 | 已初步提炼 |
| 05 | [[05_failure_driven_optimization_and_pareto\|失败驱动优化与 Pareto 多样性]] | `optim/`、`eval/`、`examples/optimization/generalization.py` | 如何从失败轨迹改进 Agent，又不让均分抹掉稀有能力 | 已初步提炼 |
| 06 | [[06_durable_human_in_the_loop\|Durable Human-in-the-loop]] | `context.py`、`agents/_agent.py`、`webui/_execution.py` | 审批如何成为可持久化、可恢复、不会重复执行的协议 | 已初步提炼 |

## 3. 明确跳过的内容

| 原主题 | 处理 | 原因 |
|---|---|---|
| 四个独立决策 | 跳过 | 可以还原为模型可见信息、期望行为和外层控制权 |
| Shared transcript / context projection | 跳过 | Pi / Hermes 已覆盖，且属于已有实践经验 |
| 基础 termination / budget | 跳过 | 组合条件和兜底上限本身增量有限 |
| Plan-based orchestration | 跳过 | 当前实现偏教学，缺少真实产品中的计划失效和恢复压力 |
| 普通多智能体评估 | 合并进 05 | 只保留失败轨迹、硬约束 judge、泛化和优化反馈闭环 |
| 框架横向示例 | 跳过 | 主要是同一模式的 API 翻译 |

## 4. 后续共同阅读方式

不再严格逐章推进。每一块只围绕一个难点讨论：

1. 先给出会在真实运行中出错的具体场景；
2. 再看仓库代码怎样表达或遗漏这个场景；
3. 最后推导生产系统需要的不变量和协议；
4. 用户确认有增量后，再更新对应笔记。
