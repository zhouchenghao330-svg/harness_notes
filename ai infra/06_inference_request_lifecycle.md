# 组件 06：推理请求生命周期与指标

> 状态：待共同阅读

## 1. 学习目标与边界

建立单实例推理请求从到达到完成的统一时间线，重点阅读第 8 章 8.1。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
到达 → 排队 → 准入 → Prefill → 首 token → 多轮 Decode → 完成 → 回收
```

## 4. 关键输入、输出和状态

待共同阅读后补充。

## 5. 公式与数量级

重点拆解 TTFT、输出间隔、请求总耗时、权重/KV/工作区容量与单实例吞吐。

## 6. 正文、代码与实验入口

- `manuscripts/08-推理优化.md`：8.1；
- `calculations/src/infra_calc/topics/decode_budget.py`；
- `calculations/src/infra_calc/topics/memory_concurrency.py`；
- `experiments/ch08/08-01/README.md`。

## 7. 与 Agent Harness 的连接

一次 Agent Run 会产生多条模型请求；每条请求的排队和生成时间共同进入 Agent 的关键路径。

## 8. 共同阅读时要回答的问题

- 首 token 和完整完成分别由哪些阶段决定？
- 权重、KV 和工作区的生命周期有何不同？
- 用户侧时间和 GPU 执行时间为什么不相等？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

多实例与多 GPU 放到后续按需回查第 9 章。

