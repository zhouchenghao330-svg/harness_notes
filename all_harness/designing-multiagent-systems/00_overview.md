# Designing Multi-Agent Systems 拆解总览

> 当前状态：已完成全仓库筛选；不再按原书式目录顺序阅读，只保留四个有工程增量的主题。
>
> 阅读方式：代码用于暴露系统边界和反例，重点讨论可以迁移到真实 Agent 产品的设计。

## 1. 对仓库的最终定位

这个仓库不是成熟 Agent 产品的生产实现，而是教学型框架和示例集合。

仓库中的大部分内容，包括普通 Agent Loop、Tool、Memory、Context、Structured Output、基础 Orchestration 和框架横向翻译，已经被 Pi / Hermes 笔记覆盖，而且 Hermes 更接近真实 Runtime，因此不再逐章阅读。

本项目只抽取四类不能被“给模型看什么、让模型做什么”直接概括的系统问题：

1. DAG 的节点激活、汇合和并发确定性；
2. Checkpoint、重放、幂等与外部副作用；
3. 失败驱动的 Agent 优化与 Pareto 多样性；
4. Durable Human-in-the-loop：审批、暂停和恢复。

已共同阅读的 [[02_multiagent_forms]] 作为背景材料保留，但不再沿原来的 03–09 教学提纲继续展开。

## 2. 为什么收缩阅读范围

下面这些主题本质上都可以还原为同一个 Agent 设计问题：

> 模型能看到什么信息，以及我们要求它产生什么行为或产物。

因此不再单独拆解：

- 四个独立决策：选谁、上下文、合并、停止；
- Shared transcript 与 context projection；
- 基础 termination、budget 和 max iterations；
- 普通 plan-based orchestration；
- Memory、Middleware 和 Structured Output；
- 不同 Agent 框架对相同模式的语法翻译。

这些并非错误，而是对已有 Agent 开发经验的重新命名，继续阅读的边际收益很低。

## 3. 新的核心地图

~~~text
真实的多执行者系统
│
├─ DAG 执行语义
│   ├─ topology fan-out ≠ runtime activation
│   ├─ control fan-in ≠ data fan-in
│   ├─ 未激活分支与 SKIPPED
│   └─ 并发完成顺序与确定性 reducer
│
├─ Durable Execution
│   ├─ checkpoint 保存逻辑状态
│   ├─ crash window 与 at-least-once execution
│   ├─ 幂等键、去重和副作用记录
│   └─ graph / prompt / tool / artifact 版本兼容
│
├─ Evaluation-driven Optimization
│   ├─ 失败轨迹 → 诊断 → 局部修改
│   ├─ minibatch gate → full evaluation
│   ├─ 保留不同能力生态位
│   └─ held-out 集验证是否真正泛化
│
└─ Durable Human-in-the-loop
    ├─ approval request 与原始 tool call 绑定
    ├─ 进程退出后仍能恢复
    ├─ 审批重复投递不导致重复副作用
    └─ 审批后续跑，不重新让模型生成参数
~~~

## 4. 新的阅读顺序

1. [[03_dag_activation_join_and_concurrency]]
2. [[04_checkpoint_replay_and_idempotency]]
3. [[05_failure_driven_optimization_and_pareto]]
4. [[06_durable_human_in_the_loop]]

详细状态和源码入口见 [[01_components]]。

## 5. 阅读判断标准

每个主题只保留三层内容：

### Repository Evidence

代码当前实际上怎样调度、保存或恢复。

### Exposed Boundary

当前实现中哪些不一致或缺失暴露了真正的系统边界。

### Derived Production Design

如果把这个能力放进真实用户场景，需要补充哪些协议、状态和不变量。

## 6. 与 Hermes 的关系

Hermes 仍然是更值得持续拆解的 Agent Runtime 样本。本仓库的用途不是替代 Hermes，而是提取 Hermes 阅读中不一定集中出现的几个专项问题：

- 静态 DAG 调度；
- durable execution；
- evaluation-driven optimization；
- 人工审批跨运行恢复。

四块讨论结束后，本仓库即可停止继续通读。
