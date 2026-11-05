# 组件 10：推测解码

> 状态：待共同阅读

## 1. 学习目标与边界

理解草稿、验证、接受和回退如何在保持目标分布的前提下减少每个输出 token 的时间，重点阅读第 8 章 8.5。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
草稿来源生成多个候选 token
  ↓
目标模型一次验证多个位置
  ↓
接受前缀并在首个拒绝位置回退
  ↓
提交本轮输出
```

## 4. 关键输入、输出和状态

重点记录 Draft Length、接受率、验证位置、每轮耗时、输出 token 数和额外显存。

## 5. 公式与数量级

共同阅读时拆解每轮平均输出 token 数与每 token 期望耗时。

## 6. 正文、代码与实验入口

- `manuscripts/08-推理优化.md`：8.5；
- `calculations/src/infra_calc/topics/speculative_round.py`；
- `calculations/src/infra_calc/topics/speculative_budget.py`；
- `calculations/src/infra_calc/topics/speculative_sampling.py`；
- `experiments/ch08/08-05/README.md`。

## 7. 与 Agent Harness 的连接

Agent 的模型等待可能被推测解码缩短，但额外模型、显存和并发竞争也会改变服务容量。

## 8. 共同阅读时要回答的问题

- 为什么验证多个位置不一定比普通一步 Decode 贵很多？
- 接受率如何影响收益？
- 草稿更长为什么不一定更快？
- 单请求加速和高并发服务收益是否一致？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

暂不展开所有推测算法变体。

