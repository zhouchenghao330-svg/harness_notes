# TextGrad：文本梯度优化范式

**发表**: Nature (2025.3.19)
**论文**: https://www.nature.com/articles/s41586-025-08661-4 | arXiv: https://arxiv.org/abs/2406.07496
**仓库**: https://github.com/zou-group/textgrad
**文档**: https://textgrad.readthedocs.io/en/latest/
**安装**: `pip install textgrad` 或 `conda install conda-forge::textgrad`

---

## 核心思想

把 **prompt / 答案 / 代码** 等**离散文本**当作"可微参数"，用 LLM 当"梯度引擎"，实现类似 PyTorch 自动微分的优化循环。

> 不是数学意义上的梯度，而是一种**范式类比**——用 LLM 的批评能力替代导数计算，用 LLM 的改写能力替代参数更新。

## API 设计（对标 PyTorch）

| PyTorch | TextGrad |
|---------|----------|
| `nn.Parameter(tensor)` | `tg.Variable("text")` |
| `loss_fn(pred, target)` | `tg.TextLoss("evaluate...")` |
| `loss.backward()` | `loss.backward()`（LLM 给反馈） |
| `optimizer.step()` | `optimizer.step()`（LLM 改文本） |
| `torch.no_grad()` | - |

## 能优化什么

**任何文本**——不限于 prompt：

- **LLM 输出的答案**：一道推理题答错了 → 自动修正
- **System Prompt**：从"你是一个助手" → 优化成包含具体约束的版本
- **代码片段**：改得更高效/正确
- **分子结构**（SMILES）：优化化学性质
- **损失函数本身**：用 LLM 定义和优化 loss
- **多模态输入**：配合 vision LLM 优化图像描述

## 背后的直觉

传统 prompt 工程是手调 → 看结果 → 再手调。TextGrad 把这个循环**自动化**了：

```
[旧 prompt] → LLM 执行 → [输出]
                         ↓
                  LLM(批评): "这个问题是因为你没考虑到XX边界条件"
                         ↓
                  LLM(修改): 根据批评重写 prompt
                         ↓
                  [新 prompt] → 循环直到收敛
```

两步各自一个 LLM 调用：
1. **Gradient Engine（批评者）**：只给反馈，不改文本
2. **Optimizer（修改者）**：根据反馈实际改写

## 为什么这是个新范式

传统上**离散文本不可微**，所以调 prompt 要么靠人力、要么靠暴力枚举（如 DSPy 的 random search）。TextGrad 把"梯度下降"这个深度学习最核心的思想**映射到了文本空间**，这意味着：

- 可以用**反向传播的思维**设计 Agent 系统——哪个环节的 prompt 影响最终结果，就在哪个环节设 Variable
- 支持**多 Variable 联合优化**——多个 Agent 的 prompt 可以一起调，自动平衡相互影响
- 可以搭出**端到端可优化的 Agent 系统**——不像传统方式强依赖人工试错

## 相关应用

- **MANGO 多智能体流网络**：在 RL 循环中嵌套 TextGrad，同时对多个 Agent 节点做文本梯度更新 — [[MANGO 多智能体流网络]]
- **单 prompt 自优化**：把 system prompt 包成 Variable，几行代码自动调优
- **代码优化**：把代码段当 Variable，定义测试通过率为 loss
- **RAG prompt 优化**：优化检索 prompt + 生成 prompt 联合调

## 局限性

- 每一轮优化都要多次调用 LLM，**成本比手调高**
- 梯度引擎的质量依赖底层 LLM 的判断力
- 没有严格的收敛保证（不同于数学梯度下降）
- 对长链系统，梯度信号可能在前面的节点衰减（这就是 MANGO 引入局部梯度 + RL 来解决的）
