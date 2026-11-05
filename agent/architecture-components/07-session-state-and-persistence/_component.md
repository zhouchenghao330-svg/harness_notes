# 07 Session State And Persistence

## 这个组件研究什么

agent 如何保存会话状态，而不是如何保存“长期记忆”。

## 应该回答的问题

- session / message 存在哪里
- branch / resume / reset / compress 怎么实现
- lineage 如何表达
- state store 是 JSONL、SQLite、Postgres 还是别的
- 哪些状态是运行期临时态，哪些是持久态

## 典型关注点

- state.db
- session metadata
- message store
- parent_session_id

## 已整理的框架笔记

- [[agent/architecture-components/07-session-state-and-persistence/hermes|Hermes：会话状态与持久化]]
- [[agent/architecture-components/07-session-state-and-persistence/langgraph|LangGraph Checkpointer：三张表、状态快照与故障恢复]]
