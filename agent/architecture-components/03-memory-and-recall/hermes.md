# Hermes

## 总体判断

Hermes 的 memory / recall 不是一个单体模块，而是至少拆成三层：

1. built-in persistent memory
2. session recall
3. background review / memory curation

其上还可以外挂一个 external memory provider 作为增强层。

所以它不是“一个 memory 系统”，而是“多层长期信息系统”。

## 1. Built-in persistent memory

### 载体

- `~/.hermes/memories/MEMORY.md`
- `~/.hermes/memories/USER.md`

### 作用

- `MEMORY.md`：环境、项目、约定、经验、durable facts
- `USER.md`：用户偏好、沟通方式、工作习惯、人格画像

### 完整链路

1. agent 启动时读取配置里的 `memory_enabled` / `user_profile_enabled`
2. 只要其中一个开启，就创建 `MemoryStore`
3. `load_from_disk()` 读取 `MEMORY.md` / `USER.md`
4. 做去重和 threat scan
5. 生成 frozen snapshot
6. snapshot 注入 system prompt
7. 运行中通过 `memory` tool 做 `add / replace / remove`
8. 写入会立刻落盘，但不会热更新当前 session 的 system prompt
9. 下次 session 启动时再重新加载新的 snapshot

### 核心设计点

- **frozen snapshot**
  不是 live memory 热更新进 prompt，而是 session 开始时冻结。
- **prompt cache 优先**
  中途写 memory 不改 system prompt，避免 prefix cache 漂移。
- **本地文件化**
  默认不是云端黑盒。

### 写入来源

同一套存储层，但有两种写入触发：

1. **前台主 agent 即时写入**
   对话过程中意识到“这条值得长期保存”，直接调用 `memory` tool。
2. **后台 reviewer 补写**
   在一定 cadence 下，background review agent 回看最近对话，再决定是否调用 `memory` tool 补写。

关键点：

- 不是两套 memory 系统
- 而是同一 built-in memory 存储层上的两种写入决策机制

## 2. Session recall

### 载体

- 本地 SQLite `state.db`
- `messages`
- `sessions`
- FTS5 索引表

### 作用

- 保留完整历史对话
- 按需从历史 session 中回忆细节
- 不把所有长期信息都压进 `USER.md / MEMORY.md`

### 完整链路

1. Hermes 把历史会话和消息落到 `state.db`
2. `messages` 通过 trigger 自动同步进 FTS5 索引
3. 当前对话需要回忆历史时，agent 调用 `session_search`
4. `session_search` 进入 discovery / scroll / browse 三种模式之一
5. 从历史 session 中返回原始消息窗口，而不是 LLM 摘要
6. agent 基于这些历史内容继续回答，必要时再决定是否写入 built-in memory

### 检索底层到底搜什么

FTS 索引主要拼接：

- `message.content`
- `tool_name`
- `tool_calls`

注意：

- 底层索引范围包含 tool message content
- 但 `session_search` 默认 discovery 会用 `role_filter = ["user", "assistant"]`
- 所以默认 recall 结果还是偏“用户和 assistant 的会话内容”，而不是工具噪声

### 三种调用形态

1. **discovery**
   给 `query`，做 FTS5 检索，找到候选 session 和命中点。
2. **scroll**
   给 `session_id + around_message_id`，围绕锚点继续翻窗口。
3. **browse**
   不给 query，先按时间浏览最近 session。

### 质量保证机制

Hermes 的 recall 质量不是靠“上游语义提炼”，而是靠“原文检索 + 工程型包装”：

1. **query sanitization**
   先清洗 FTS5 查询语法，避免 MATCH 报错和错误切分。
2. **CJK fallback**
   中文走 trigram FTS5，短 CJK 查询退回 LIKE。
3. **snippet + local context**
   不只给命中词，还给前后局部上下文。
4. **anchored view**
   围绕命中 message 提供可继续翻阅的窗口。
5. **bookends**
   额外给 session 开头和结尾片段，帮助理解整段 session 的主题和收束状态。
6. **lineage dedupe**
   沿 `parent_session_id` 处理 lineage，避免 branch / compress 后的重复召回。
7. **exclude current lineage**
   尽量排除当前 session lineage，防止 recall 退化成“把当前上下文再搜一遍”。
8. **default role filtering**
   默认更偏 user / assistant，不让工具输出噪声主导 recall。

### 关键判断

- session recall 是 memory 的补集，不是替代品
- 它偏工程型原文检索，而不是高层语义用户建模
- 默认不依赖 LLM summarization

## 3. Background review / memory curation

### 作用

负责把“最近对话里值得长期保留的内容”筛出来。

### 完整链路

1. 一轮主对话结束
2. 如果到达 memory review cadence，则触发 background review
3. fork 一个 review agent
4. review agent 回看 conversation snapshot
5. 判断有没有值得沉淀进 built-in memory 的内容
6. 如有必要，调用同一个 `memory` tool 写入

### 本质

- 存储层依然是本地 built-in memory
- 但“哪些内容值得沉淀”已经是模型驱动的语义判断

### 关键意义

这层把 Hermes 从“文件型 memory”推进到“主动积累型 memory”。

如果只看 `MEMORY.md / USER.md`，会误以为它只是 memory CRUD；
真正有 agent 味道的部分，在于 reviewer 对记忆写入的补采样与补整理。

## 4. External memory provider

### 定位

- additive enhancement
- 不是 built-in memory 的替代
- 同时只能启用一个 external provider

### 作用

在 built-in memory 之外，增加：

- 语义检索
- 用户建模
- 知识图谱
- 更强的事实存储 / synthesis / reasoning

### 两类路线

1. **service-backed provider**
   代表：Honcho、Mem0、Supermemory  
   特点：更强用户建模，可能依赖云服务或自托管后端。

2. **local provider**
   代表：Holographic  
   特点：本地 SQLite / FTS5 / trust scoring，更适合单机研究。

## 总结

Hermes 在 memory / recall 上的设计哲学不是“一个大而全的长期记忆黑盒”，而是：

- 小而常驻的 built-in memory
- 大而按需检索的 session recall
- 模型驱动的 background review
- 可插拔的 external provider 增强层

这是一个明显分层的长期信息系统，而不是单一 memory API。
