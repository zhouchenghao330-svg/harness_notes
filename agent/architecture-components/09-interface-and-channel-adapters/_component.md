# 09 Interface And Channel Adapters

## 这个组件研究什么

agent 如何接不同的人机交互入口。

## 应该回答的问题

- CLI、TUI、Web、Telegram、Discord 等入口怎么统一
- 不同 channel 的 session 怎么映射
- UI 层和 agent runtime 的边界是什么
- 中断、流式输出、附件、富媒体在不同通道怎么适配

## 典型关注点

- cli
- tui
- gateway
- platform adapters
