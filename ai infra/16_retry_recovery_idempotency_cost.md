# 组件 16：Retry、恢复、幂等与完整任务成本

> 状态：待共同阅读

## 1. 学习目标与边界

理解失败后的重做、续接、模型升级、状态恢复和副作用去重，重点阅读第 11 章 11.5。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
执行操作
  ↓
结果或确认丢失 / 超时 / 进程失败
  ↓
读取持久化状态与操作身份
  ├─ 已完成：返回原结果
  ├─ 可续接：从检查点继续
  ├─ 可安全重试：重新执行
  └─ 不确定副作用：停止并核对
```

## 4. 关键输入、输出和状态

重点记录 Operation ID、提交状态、结果记录、Checkpoint、外部副作用、重试分支和失败成本。

## 5. 公式与数量级

共同阅读时沿有限执行树计算成功概率、截止前成功率、重复工作和期望成本。

## 6. 正文、代码与实验入口

- `manuscripts/11-资源调度与运行环境.md`：11.5；
- `calculations/src/infra_calc/topics/retry_paths.py`；
- `calculations/src/infra_calc/topics/operation_ordering.py`；
- `calculations/src/infra_calc/topics/protocol_retry.py`；
- `calculations/src/infra_calc/topics/kv_restore.py`；
- `experiments/ch11/11-04/README.md`；
- `experiments/ch11/11-09/README.md`。

## 7. 与 Agent Harness 的连接

这是 Session Persistence、Tool Retry、Crash Recovery 和外部副作用安全真正落到系统层的位置。

## 8. 共同阅读时要回答的问题

- 工具成功但 ACK 丢失时为什么不能直接重试？
- 幂等键、持久化结果和 Checkpoint 分别解决什么问题？
- 从头重做、局部续接和更换模型如何比较？
- 失败尝试、重复计算和驻留资源如何计入完整任务成本？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

不同外部系统的幂等协议需要在具体工具场景中单独确认。

