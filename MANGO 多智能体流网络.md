# MANGO: Multi-Agent Network Gradient Optimization

**来源**: openJiuwen（华为泊松实验室/维纳研究所）
**论文**: [Reinforced Collaboration in Multi-Agent Flow Networks](https://arxiv.org/abs/2605.12943) (arXiv:2605.12943)
**代码**: [TextGrad](https://github.com/zou-group/textgrad) ← 底层的 prompt 优化框架
**链接**: [机器之心文章](https://mp.weixin.qq.com/s/Eleh8ZlAVYpgV_4MCq_2vQ)

## 核心思想

MANGO 是一个**数据驱动的多 Agent 协作优化框架**——从历史成功工作流中学习，用 RL + 文本梯度联合优化协作路径和 Agent 行为。

## 解决的问题

多智能体协作的**错误传播**——一个 Agent 幻觉了、路径走歪了，错误沿着协作链往后传。现有方法（AutoGen、MetaGPT）依赖人工配置或启发式搜索，上限卡在搜索策略设计上。

## 架构

### 三步走

**① 流网络构建**
- 把 N 条历史成功工作流的每个操作按语义相似度聚类成节点
- 每个节点 = 一个 **system prompt + role description**，即一个 LLM Agent
- 相邻操作不放同一节点，保持工作流转换完整性

**② RL 选路径（REINFORCE 算法）**
- 新任务来了，策略网络 π_φ 从源点到汇点动态选出一条协作路径
- 状态 = 当前节点与邻节点的向量相似度
- 奖励 = 过程正确性 + 最终结果加权

**③ 文本梯度优化节点 prompt**
- **全局梯度**：在 sink 节点比较 ground-truth，得出整体反馈
- **局部梯度**：在每个中间节点，拿输出跟历史真实中间步骤比较
- 公式：∇ΘᵢL = ∇globalL_final + ∇localLᵢ
- 消融实验：去掉局部梯度，准确率从 58.44% → 54.73%（掉 6.8%）

### 文本梯度到底是什么

不是数学梯度，而是用 **LLM 当"梯度引擎"**（TextGrad 框架）：

- **Gradient Engine（批评者）**: "你犯了 XX 错误，因为...建议修改 prompt..."
- **Optimizer（修改者）**: 根据批评改写 prompt

### 节点跳跃（Skip-K）

- 不是"判断节点优化好了再跳"，而是 RL **动作空间里允许跳 K 步**
- K=1（默认）= 不跳，走一步算一步
- K=3：省 21.5% 成本 + 24.1% 时间，但准确率掉 10.6%
- 跳过的节点用历史工作流的真实中间步骤填充

## 实验结果

7 个数据集（HumanEval、MBPP、MATH500、GSM8K、DROP、MMLU、GPQA-Diamond），GPT-4o-mini 作为基座：
- MATH500 比 MaAS **高 12.8%**
- DROP 比 AFlow **高 5.1%**
- Skip-3 下训练时间缩短 **41.5%**，推理时间缩短 **47.4%**

## 与 LangGraph 的区别

| | LangGraph | MANGO |
|---|---|---|
| 节点 | 人工定义固定语义角色 | 从历史轨迹聚类出来的 action 簇 |
| 拓扑 | 开发者硬编码 DAG | 动态建图，边权重由 RL 训练 |
| 路径 | 固定 | RL 动态选择 |

## 启发

- **协作架构比模型大小更重要**——GPT-4o-mini 刷出 SoTA
- **RL + 文本梯度的互相增强循环**：改 prompt → 影响状态 → 影响路径选择 → 更多训练信号 → 进一步优化 prompt

## TextGrad（底层框架）

TextGrad 是 MANGO 底层的 prompt 优化框架，**跟工作流无关**，是一个通用工具：
- 可以优化：LLM 答案、system prompt、代码片段、分子结构字符串……**任何需要 LLM 调用且有 system_prompt 的东西**
- API 设计类似 PyTorch：`Variable` → `loss_fn` → `backward()` → `optimizer.step()`
- 已发表在 **Nature** (2025.3)
- 仓库：https://github.com/zou-group/textgrad

## 局限性（论文自述）

1. 流网络构建依赖历史工作流的质量和覆盖度
2. 超参数多（相似度阈值 δ、Skip-K），需要调
3. RL + 文本梯度联合优化增加系统复杂度
