# 组件 08：KV Cache 与 Prefix 复用

> 状态：待共同阅读

## 1. 学习目标与边界

理解 KV Cache 的分配、分页、共享、写时复制、淘汰与释放，重点阅读第 8 章 8.3。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
输入 token
  ↓
Prefill 生成 KV Block
  ↓
Block Table 映射逻辑块与物理块
  ↓
Decode 持续读取并追加 KV
  ↓
共享 / 复制 / 淘汰 / 释放
```

## 4. 关键输入、输出和状态

重点记录 Block Size、逻辑/物理映射、引用关系、共享前缀、活跃序列和回收时机。

## 5. 公式与数量级

共同阅读时计算单 token KV 大小、完整 Context 占用、共享节省量和重算/换出交点。

## 6. 正文、代码与实验入口

- `manuscripts/08-推理优化.md`：8.3；
- `calculations/src/infra_calc/topics/kv_pages.py`；
- `calculations/src/infra_calc/topics/kv_trace.py`；
- `calculations/src/infra_calc/topics/apc_trace.py`；
- `calculations/src/infra_calc/topics/prefix_value.py`；
- `experiments/ch08/08-03/README.md`；
- `experiments/ch08/08-04/README.md`。

## 7. 与 Agent Harness 的连接

System Prompt、工具说明、多轮历史和分支 Agent 都可能形成可复用前缀，也会延长 KV 的驻留时间。

## 8. 共同阅读时要回答的问题

- 应用层 Context 在服务端如何变成 KV？
- Prefix Cache 命中到底省掉了什么？
- 分支共享为什么需要写时复制？
- Cancel 后 KV 何时真正释放？
- 命中率高为什么仍可能没有端到端收益？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

不同 Attention 架构的 KV 形状按需回查第 2 章。

