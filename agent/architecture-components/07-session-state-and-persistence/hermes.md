# Hermes

## 这个组件里的 Hermes 重点

在 session state / persistence 这一层，Hermes 的关键不是 memory，而是：

- `state.db` 作为会话状态库
- `sessions` / `messages` 两张核心表
- `parent_session_id` 表达 lineage
- `session_search`、`resume`、`branch`、`compress` 等能力都建立在这层之上

## 核心判断

- built-in memory 和 session state 是两套东西
- `MEMORY.md / USER.md` 解决的是 durable facts
- `state.db` 解决的是“完整会话如何保存、回放、分叉、检索”

## 数据形态

### sessions

保存：

- session 元数据
- source
- model
- started_at / ended_at
- message_count
- title
- `parent_session_id`

### messages

保存：

- `session_id`
- `role`
- `content`
- `tool_call_id`
- `tool_calls`
- `tool_name`
- timestamp
- reasoning / finish_reason 等附加字段

## 为什么这层重要

Hermes 的很多“像长期记忆一样”的能力，其实底座并不在 memory，而在 session state：

- `session_search`
- `browse`
- anchored scroll
- lineage dedupe
- current lineage exclusion

这些都依赖 `state.db`，不是依赖 `MEMORY.md / USER.md`。

## 与 recall 的关系

`session recall` 是建立在 session persistence 之上的上层能力。

也就是说：

- `07-session-state-and-persistence`
  关注：会话怎么存、怎么表达 lineage、怎么维护消息级状态
- `03-memory-and-recall`
  关注：这些状态如何被当作长期信息重新找回和利用

## 当前已确认的技术点

- SQLite + WAL / fallback
- `messages_fts`
- `messages_fts_trigram`
- trigger 自动同步 FTS 索引
- `parent_session_id` 链路支撑 lineage 相关逻辑

## 后续可继续拆

- `state.db` schema 全量梳理
- `branch / compress / resume` 对 session lineage 的影响
- tool / reasoning / multimodal content 在 message store 中的具体编码方式
