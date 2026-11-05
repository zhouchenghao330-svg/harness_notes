# 组件 14：资源池、抢占与调度

> 状态：待共同阅读

## 1. 学习目标与边界

理解异构资源、成组放置、优先级、抢占、验证长尾和资源回收，重点阅读第 11 章 11.3。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
任务进入队列
  ↓
检查 GPU / CPU / 内存 / 网络组合约束
  ↓
放置、等待或抢占
  ↓
执行与恢复
  ↓
完成确认后回收资源
```

## 4. 关键输入、输出和状态

重点记录成组资源、优先级、迁移成本、已完成前缀、验证 Barrier 和回收信号。

## 5. 公式与数量级

待共同阅读后补充。

## 6. 正文、代码与实验入口

- `manuscripts/11-资源调度与运行环境.md`：11.3；
- `calculations/src/infra_calc/topics/completion_reclaim.py`；
- `calculations/src/infra_calc/topics/feedback_queue.py`；
- `calculations/src/infra_calc/topics/topology_allocation.py`；
- `experiments/ch11/11-03/README.md`；
- `experiments/ch11/11-04/README.md`；
- `experiments/ch11/11-05/README.md`。

## 7. 与 Agent Harness 的连接

Cancel、Pause、Retry 和 Subagent 并发最终都会转化为资源池中的放置、抢占、恢复和回收事件。

## 8. 共同阅读时要回答的问题

- 有空闲 GPU 为什么任务仍可能放不进去？
- 抢占后从头重做和保留前缀如何比较？
- 返回超时为什么不代表计算已经停止？
- 验证长尾为什么会延迟整个 Batch 的释放？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

多数据中心调度不在第一轮范围内。

