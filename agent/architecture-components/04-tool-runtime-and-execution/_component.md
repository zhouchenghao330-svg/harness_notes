# 04 Tool Runtime And Execution

## 这个组件研究什么

agent 怎么把“语言决策”落到可执行工具上。

## 应该回答的问题

- tool schema 从哪里来
- tool call 怎么路由
- 工具返回值如何回流给模型
- 本地命令、文件编辑、网络工具、浏览器工具怎么接入
- tool runtime 如何处理失败、超时、重试

## 典型关注点

- tool registry
- tool dispatch
- shell / patch / file / browser execution
- tool result normalization
