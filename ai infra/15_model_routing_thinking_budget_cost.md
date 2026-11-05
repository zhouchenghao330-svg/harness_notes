# 组件 15：模型路由、Thinking Budget 与成本

> 状态：待共同阅读

## 1. 学习目标与边界

理解模型选择、思考预算、缓存用量、排队、限流和计价如何共同影响成功任务成本，重点阅读第 11 章 11.4。

## 2. 先用直白的话讲

待共同阅读后补充。

## 3. 最小运行链

```text
任务与质量要求
  ↓
选择模型、服务路径和 Thinking Budget
  ↓
发生输入、缓存输入、输出和失败尝试
  ↓
检查质量与时限
  ↓
计算有效完成成本
```

## 4. 关键输入、输出和状态

重点记录任务质量、Token 用量、缓存命中、排队时间、服务单价、失败率和截止时间。

## 5. 公式与数量级

共同阅读时区分每次请求价格与每个成功任务的期望成本。

## 6. 正文、代码与实验入口

- `manuscripts/11-资源调度与运行环境.md`：11.4；
- `calculations/src/infra_calc/topics/routing_cost.py`；
- `calculations/src/infra_calc/topics/strategy_record_cost.py`；
- `case-studies/routing-cost-and-completion.md`；
- `experiments/ch11/11-07/README.md`；
- `experiments/ch11/11-08/README.md`。

## 7. 与 Agent Harness 的连接

Harness 的模型路由、Reasoning 配置和 Context 复用策略决定真实调用用量，但最终评价必须回到任务是否正确且按时完成。

## 8. 共同阅读时要回答的问题

- 更便宜的模型为什么可能让成功任务更贵？
- Thinking Token 增加如何改变质量和时间？
- 缓存字段、输入字段和输出字段如何避免重复计数？
- 自建服务、按量 API 和订阅为什么不能只比 token 单价？

## 9. 已确认结论

待共同阅读后补充。

## 10. 待验证与暂不展开

具体产品价格会变化，后续若引用必须重新核对；当前先学习计算结构。

