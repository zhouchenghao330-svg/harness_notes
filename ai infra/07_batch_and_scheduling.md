# 组件 07：Batch、连续批处理与调度

> 状态：待共同阅读

## 1. 学习目标与边界

理解 Batch 为什么提高吞吐，以及固定 Batch、连续批处理和分块 Prefill 如何改变等待，重点阅读第 8 章 8.2。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
等待请求集合
  ↓
调度器选择本轮 token 预算
  ├─ 新请求 Prefill
  └─ 活跃请求 Decode
  ↓
执行一轮
  ↓
完成请求离开，新请求加入
```

## 4. 关键输入、输出和状态

重点记录活跃序列、等待队列、Token Budget、Batch Shape 和每轮完成请求。

## 5. 公式与数量级

待共同阅读后补充。

## 6. 正文、代码与实验入口

- `manuscripts/08-推理优化.md`：8.2；
- `calculations/src/infra_calc/topics/iteration_batching.py`；
- `calculations/src/infra_calc/topics/service_replay.py`；
- `calculations/src/infra_calc/topics/batch_reuse.py`；
- `experiments/ch08/08-01/README.md`；
- `experiments/ch08/08-02/README.md`。

## 7. 与 Agent Harness 的连接

多 Agent、多轮调用和长短请求混合会进入同一服务队列，调度策略会反过来改变每个 Agent 的等待和响应节奏。

## 8. 共同阅读时要回答的问题

- Batch 复用的是什么？
- 吞吐提高时单请求延迟为什么可能变差？
- 长 Prefill 为什么会干扰正在 Decode 的请求？
- Token Budget 如何影响公平性和执行形状？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

暂不讨论跨实例全局调度。

