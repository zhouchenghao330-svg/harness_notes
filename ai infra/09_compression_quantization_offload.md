# 组件 09：压缩、量化与 Offload

> 状态：待共同阅读

## 1. 学习目标与边界

理解权重/KV 压缩和分层存储如何交换容量、质量、带宽与延迟，重点阅读第 8 章 8.4。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
状态太大或放不下
  ├─ 降低位宽 / 改变编码
  ├─ 把冷状态换出到主机或远端
  └─ 需要时预取或重算
```

## 4. 关键输入、输出和状态

重点记录编码位宽、元数据、链路带宽、预取窗口、命中率、质量门槛和工作区。

## 5. 公式与数量级

待共同阅读后补充。

## 6. 正文、代码与实验入口

- `manuscripts/08-推理优化.md`：8.4；
- `calculations/src/infra_calc/topics/kv_codec.py`；
- `calculations/src/infra_calc/topics/kv_quality.py`；
- `calculations/src/infra_calc/topics/kv_tiers.py`；
- `case-studies/weight-offload-execution.md`；
- `experiments/ch08/08-07/README.md`；
- `experiments/ch08/08-08/README.md`。

## 7. 与 Agent Harness 的连接

长 Context 和大量并发 Session 会迫使服务在保留、压缩、换出、重算和拒绝请求之间选择。

## 8. 共同阅读时要回答的问题

- 压缩节省的容量由哪些元数据抵消？
- Offload 为什么不是“显存不够就放内存”这么简单？
- 预取什么时候能隐藏搬移时间？
- 质量门槛如何进入系统配置选择？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

不把单一量化格式的结果推广到所有模型。

