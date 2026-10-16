# 组件 05：失败驱动的 Agent 优化与 Pareto 多样性

> 状态：已完成第一轮提炼，待共同深入
>
> 核心判断：Agent 优化不是让模型泛泛反思并重写 Prompt，而是把失败轨迹变成候选修改，再用受控评估决定哪些能力方向值得保留。

## 1. 优化循环

仓库把不同 optimizer 归纳成同一个骨架：

~~~text
seed candidate
   ↓
在数据集上执行 agent rollout
   ↓
judge 对每个任务评分并给出 reasoning
   ↓
选择弱项和失败轨迹
   ↓
reflector 诊断并提出结构化修改
   ↓
minibatch gate
   ↓ 通过
full evaluation
   ↓
selection：决定保留哪些候选
~~~

Optimizer 真正不同的主要是：

- mutate 什么；
- 根据什么反馈 mutate；
- 怎样选择存活候选；
- 下一轮从哪些 parent 继续生成。

## 2. 失败轨迹必须包含什么

只保留最终分数无法支持有效诊断。至少需要：

- task identity 和原始输入；
- agent 最终输出；
- 必要时的完整消息和 tool calls；
- 每个评分维度；
- judge reasoning；
- execution success / error；
- agent usage 和 judge usage；
- 当前 candidate config；
- 产生该 candidate 的 parent 和修改 rationale。

仓库的 `OptimizationTrace` 记录：

- seed / full / gate evaluation；
- parent selection；
- reflector 看到了什么和输出了什么；
- noop proposal；
- minibatch gate verdict；
- candidate 进入和 selection 后的 pool。

这使优化过程可以审计，而不仅是保存最终最佳 Prompt。

## 3. 为什么不能只保留平均分最高者

假设每个任务是一条能力维度：

| Candidate | 常规任务 | PII 任务 | 格式任务 | 平均分 |
|---|---:|---:|---:|---:|
| A | 9 | 2 | 9 | 6.7 |
| B | 7 | 10 | 7 | 8.0 |
| C | 8 | 8 | 10 | 8.7 |

只按平均分选择会很快淘汰 A、B。

但 B 可能是唯一真正学会“不得回显敏感信息”的候选。它携带的是一个稀有但关键的能力方向。如果立即淘汰，后续 mutation 只能围绕 C 的已有行为继续局部爬山。

因此多样性不是为了让答案风格丰富，而是为了保留不同的 **failure coverage**。

## 4. 仓库中的 Pareto 具体是什么

当前 `pareto_frontier()` 会保留：

> 至少在一个 task 上达到全池最高分的 candidate。

Parent selection 再按照 candidate 赢得多少任务加权采样。

这不是最严格的多目标 Pareto dominance 定义。严格定义通常是：不存在另一个候选在所有维度上都不差，并至少在一个维度上更好。

当前实现更接近 **per-task champion preservation**。名称可以商榷，但它表达了一个有价值的搜索策略：

~~~text
不要只繁殖当前均分冠军；
也要繁殖掌握了独特困难样本的候选。
~~~

## 5. Minibatch gate 的真实作用

每个候选都跑完整数据集非常昂贵，因此先用少量任务筛选：

~~~text
proposal
  ↓
cheap minibatch：判断是否值得继续投入
  ↓ pass
full evaluation：获得可比较的正式结果
~~~

这里有一个风险：如果 minibatch 没覆盖候选擅长或修复的稀有失败类型，它可能在正式评估前就被错误淘汰。

所以 gate sample 不应只做均匀随机采样。可以包含：

- parent 的已知失败样本；
- 本次修改声称要解决的目标样本；
- 防止能力退化的 regression anchors；
- 少量随机样本。

## 6. Judge 必须区分硬约束和主观质量

如果所有维度都交给 LLM Judge，优化器可能学习 Judge 的偏好和噪声，而不是业务规则。

仓库的 generalization 示例做了一件值得保留的事：

- 语气等主观标准由 LLM Judge 判断；
- 格式、正则、不得泄漏 PII 等客观约束使用确定性检查覆盖。

也就是说：

~~~text
deterministic checks：硬规则、协议、结构、安全约束
LLM judge：相关性、解释质量、语气等模糊指标
~~~

优化系统不应让概率 Judge 重新解释本来可以确定计算的规则。

## 7. 泛化验证

优化数据必须和 held-out evaluation 分开：

~~~text
TRAIN：供 reflector 看到失败并修改 candidate
TEST：优化器从未看到，只用于最后验证
~~~

需要同时看：

- train gain；
- test gain；
- generalization gap；
- 关键 failure slice 是否改善；
- 原有能力是否 regression；
- 优化成本是否超过收益。

均分提高但 held-out 下降，说明优化器可能只是在记忆评估集或迎合 Judge。

## 8. 成本必须拆开

优化一次 Agent 配置至少产生三类模型成本：

1. **Agent rollout**：候选在任务上实际运行；
2. **Judge**：对每条 trajectory 评分；
3. **Reflection / Proposal**：分析失败并提出修改。

比较 optimizer 时应使用相同 rollout budget，而不是只比较 round 数。一次 round 可能生成不同数量的候选，也可能触发不同数量的 full evaluation。

## 9. 从这里推导出的生产设计

1. 优化单位应是 versioned Agent config，而不是直接覆盖线上 Prompt；
2. 每次 proposal 必须能追溯到具体失败证据；
3. 修改面需要白名单，例如只允许 instructions、skill 或 tool catalog；
4. 硬约束使用确定性 evaluator；
5. 保留 failure slices 和 regression anchors；
6. 多样性按能力覆盖衡量，而不是按文本差异衡量；
7. 任何“最佳候选”都必须经过 held-out 和线上灰度验证；
8. 优化轨迹本身需要版本化和隐私控制。

## 10. 关键源码入口

- `picoagents/src/picoagents/optim/_base.py`
  - universal optimization loop
  - `Candidate`
  - `CostLog`
- `picoagents/src/picoagents/optim/_reflective.py`
  - weak task diagnosis
  - structured edits
- `picoagents/src/picoagents/optim/_reflective_pareto.py`
  - `pareto_frontier()`
  - weighted parent selection
- `picoagents/src/picoagents/optim/_trace.py`
  - `OptimizationTrace`
- `examples/optimization/generalization.py`
  - deterministic + LLM hybrid judge
  - train / held-out split

## 11. 一句话总结

> 可靠的 Agent 优化是一个有预算、有证据、有保留多样性策略的搜索过程；Pareto 的价值不是追求多个漂亮 Prompt，而是避免平均分把稀有但关键的能力方向过早淘汰。
