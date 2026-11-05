# 组件 12：Agent 任务的资源生命周期

> 状态：待共同阅读

## 1. 学习目标与边界

区分 Agent 的实际 CPU/GPU 工作量、墙钟时间、等待时间和环境驻留，重点阅读第 11 章 11.1。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
模型阶段
  ↓
工具阶段
  ↓
模型阶段
  ↓
验证 / 反馈
  ↓
任务完成与资源回收
```

## 4. 关键输入、输出和状态

重点区分 CPU 秒、GPU 秒、GiB·秒、环境墙钟驻留和任务完成条件。

## 5. 公式与数量级

待共同阅读后补充。

## 6. 正文、代码与实验入口

- `manuscripts/11-资源调度与运行环境.md`：11.1；
- `calculations/src/infra_calc/topics/environment_resources.py`；
- `calculations/src/infra_calc/topics/trace_resource_bridge.py`；
- `experiments/ch11/11-01/README.md`；
- `experiments/ch11/11-01/process-resources/README.md`。

## 7. 与 Agent Harness 的连接

Agent Loop 决定资源在什么时候工作、等待、保持状态和释放；Harness 的完成信号必须与真实资源生命周期区分。

## 8. 共同阅读时要回答的问题

- 工具只运行几秒，环境为什么可能驻留几十秒？
- 墙钟时间、CPU 工作量和内存占用如何分账？
- “工具返回”是否代表进程和资源已经释放？
- Agent 什么时候才算真正完成？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

具体 Sandbox 实现放到组件 13。

