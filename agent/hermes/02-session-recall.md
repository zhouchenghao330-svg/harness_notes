# 02 Session Recall

## 关注点

- Hermes 如何保存全量会话历史
- `session_search` 的检索和浏览形态
- 为什么它不把所有历史都压进 `USER.md`

## 已确认

- Hermes 把历史会话存进 SQLite。
- `session_search` 走的是本地 DB 检索，不是 LLM 摘要链。
- 检索底座是 FTS5，并支持 discovery / scroll / browse 三种形态。

## 底层到底搜的是什么

先纠正一个很容易产生的误解：

- 不是只搜“用户和 assistant 的自然语言聊天文本”。
- FTS 底层索引来自 `messages` 表，而 `messages` 表本身包含 `user` / `assistant` / `tool` 等不同角色消息。

FTS 索引时实际拼接进去的字段主要是：

- `message.content`
- `tool_name`
- `tool_calls`

因此：

- 如果某条 `tool` 角色消息本身有 `content`，它也会进入索引。
- 所以底层索引范围其实比“纯聊天”更大。

但 `session_search` 这个工具层默认又会收一层口：

- discovery 默认 `role_filter = ["user", "assistant"]`

这意味着：

- 底层索引包含 tool message content
- 但默认召回结果偏向用户和 assistant 的会话内容
- tool 噪声不会轻易主导 recall 结果

## 不做什么

Hermes 的 session recall 明显不是“先抽象，再检索”的路线。

它默认不做：

- 关键词提取
- 标签抽取
- embedding 向量召回
- LLM 先总结成 recall card 再检索

它做的是：

1. 原始消息直接写入 SQLite
2. 通过 trigger 自动同步到 FTS 索引
3. 查询时直接对原始文本层做 FTS5 / trigram / LIKE 检索

所以这一层更像：

- 原文 recall
- 工程型检索
- 低幻觉、低成本、可回溯

而不是：

- 高语义抽象后的用户建模检索

## 召回质量靠什么保证

Hermes 在 recall 层追求的不是“更聪明地猜”，而是“更稳地找”。

### 1. Query sanitization

查询不会原样塞进 FTS5。

会先做 `_sanitize_fts5_query()`，处理：

- 不配对引号
- FTS5 特殊字符
- 裸露布尔词
- 带点号、连字符、下划线的 token

作用：

- 防止 MATCH 语法炸掉
- 尽量保住用户原本的词面意图

### 2. CJK fallback

默认 `unicode61` tokenizer 对中文不好，所以 Hermes 额外做了：

- `messages_fts_trigram`
- 对太短的 CJK 查询再退回 `LIKE`

作用：

- 避免中文 phrase / substring recall 质量过差
- 避免短中文 token 在 trigram 路径上直接失效

### 3. Snippet + local context

底层 `search_messages()` 不只回一个裸命中。

它会补：

- snippet
- 命中消息前 1 条
- 命中消息后 1 条

作用：

- 提高局部可解释性
- 避免 agent 只看到一个孤立命中词就误判语义

### 4. Anchored view

`session_search` 的 discovery 在 message-level 搜索之上，还会把结果包装成 anchored session slice：

- `match_message_id`
- `messages`
- `messages_before`
- `messages_after`

作用：

- 让 recall 不是“搜到就结束”
- 而是“先命中，再围绕锚点继续翻”

### 5. Bookends

discovery 还会额外带：

- `bookend_start`
- `bookend_end`

作用：

- 让 agent 快速理解这段 session 的开头主题和结尾状态
- 防止只看局部命中窗口时，把整段历史会话的意图看窄

### 6. Lineage dedupe

Hermes 的 session 不是纯线性的，有：

- resume
- branch
- compress
- parent / child session chain

所以 discovery 会先按 lineage root 去重。

作用：

- 避免同一条历史脉络因为 session 分叉被重复命中
- 降低 recall 结果碎片化

### 7. Exclude current lineage

默认会尽量排除当前 session lineage。

作用：

- 防止 recall 退化成“把当前上下文再搜一遍”
- 保持 `session_search` 的跨会话意义

### 8. Default role filtering

虽然底层索引包含 tool message content，但默认 discovery 仍只看：

- `user`
- `assistant`

作用：

- 减少工具输出、日志碎片对历史 recall 的污染
- 让 recall 更像“我们之前说过什么”，而不是“工具曾经吐过什么”

## 这层的意义

- 把“重要但不值得常驻 prompt 的细节”留在会话库里。
- 让长期记忆分层：关键事实常驻，细节历史按需召回。
- 避免把用户画像做成一锅粥式的超长 summary。

## 关键判断

- Hermes 把 session recall 当作 memory 的补集，而不是 memory 的替代。
- 这层明显偏工程化检索，而不是偏抽象用户建模。
- 召回质量主要靠原文检索、fallback、窗口包装、lineage 去重，而不是靠上游语义提炼。

## 还要继续看的点

- `state.db` 的 schema
- `search_messages` / `get_messages_around` 的数据路径
- compaction / branching 后 lineage 怎么处理
- bookend 设计为什么必要
- tool message 什么时候值得显式放开 role_filter

## 代码入口

- `tools/session_search_tool.py`
- `hermes_state.py`
