# 08 Model Routing And Provider Abstraction

## 这个组件研究什么

agent 如何屏蔽不同模型和不同 provider 的差异。

## 应该回答的问题

- provider abstraction 在哪一层
- 主模型和辅助模型如何区分
- compression / vision / title generation 是否单独路由
- provider failover / fallback / routing 怎么做

## 典型关注点

- provider profile
- model routing
- auxiliary model path
- fallback chain
