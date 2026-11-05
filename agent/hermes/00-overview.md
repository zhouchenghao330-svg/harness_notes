# Hermes 长期记忆总览

## 目标

把 Hermes 的“长期记忆”拆成清晰的系统结构，而不是把 `memory`、`session_search`、`Honcho` 混成一个概念。

## 三层结构

### 1. Built-in persistent memory

- 载体：`MEMORY.md`、`USER.md`
- 作用：保存小而关键、需要常驻上下文的 durable facts
- 特征：本地文件、容量受限、session 启动时注入 prompt

### 2. Session recall

- 载体：`state.db` 中的历史会话和消息
- 作用：保存细节型历史，不强行压缩进常驻 prompt
- 特征：SQLite + FTS5，本地检索，按需召回

### 3. Background review / memory curation

- 载体：后台 fork 出来的 review agent
- 作用：从会话里主动筛出值得沉淀到长期记忆的内容
- 特征：存储层是本地，但“值不值得记”由模型判断

## 外挂增强层

### External memory provider

- 载体：`memory.provider`
- 作用：在 built-in 之外增加语义检索、用户建模、知识图谱、事实存储等能力
- 特征：单选 provider；可能是本地实现，也可能是云/自托管服务

## 当前判断

- 默认模式下，Hermes 的长期记忆不是云端黑盒。
- 真正接近“黑盒用户建模”的部分主要出现在外接 provider，尤其是 Honcho 这类服务型 provider。
- Hermes 的默认设计更像“本地小记忆 + 本地会话库 + LLM 驱动整理器”。

## 后续拆解问题

- `USER.md` 和 `MEMORY.md` 的边界是否长期稳定。
- memory review 的触发频率、写入策略、失败兜底是什么。
- `session_search` 为什么坚持不用 LLM summarization。
- Honcho、Holographic 这两类 provider 分别代表什么设计哲学。
