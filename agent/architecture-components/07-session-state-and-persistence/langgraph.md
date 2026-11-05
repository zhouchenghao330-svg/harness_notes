# LangGraph Checkpointer：对话落库、状态快照与故障恢复

> 整理日期：2026-10-14。
> 范围：已讨论的 Checkpointer 概念、PostgreSQL 三张核心表、消息合并与故障恢复。不是整个 LangGraph Runtime 的完整拆解。
> 证据：整理时核对官方仓库提交 `a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5`；文中 CP1、v1 等都是教学简写，表格里的消息展示的是解码后的内容。

## 1. 先建立一个整体理解

LangGraph 的 Checkpointer 保存的是 **Agent 运行到某个阶段时的状态**。聊天历史可以是状态的一部分，工具结果、其他状态字段和继续执行所需的信息也可以被保存。

可以先用下面三句话记住 PostgreSQL 实现中的分工：

| 表 | 可以怎么理解 | 回答的问题 |
| --- | --- | --- |
| `checkpoints` | 状态快照目录，带版本引用和运行元数据 | 这个会话有哪些状态快照？当前快照引用了哪些字段版本？ |
| `checkpoint_blobs` | 状态字段的数据仓库 | 某个字段在某个版本的实际值是什么？ |
| `checkpoint_writes` | 节点执行产生的中间结果记录 | 基于当前快照执行的任务，已经产生了哪些更新？ |

```text
会话 ID
   ↓
找到一份状态快照
   ├─ 读取快照内直接保存的简单值
   ├─ 根据字段版本，读取并解码 Blob
   └─ 读取这份快照关联的节点中间结果
   ↓
恢复状态，并由执行引擎决定怎样继续
```

这里的“目录”是理解方式：`checkpoints` 不是只有一个指针，也保存了实际的快照结构、部分状态值和运行元数据。

## 2. user_id、thread_id、checkpoint_id 分别是什么

| 名称 | 标识什么 | 示例 |
| --- | --- | --- |
| `user_id` | 哪个用户，由业务系统管理 | `user_100` |
| `thread_id` | 哪条会话／执行线程，可以映射业务的 conversation_id | `chat_001` |
| `checkpoint_id` | 这条会话的一份状态快照 | `cp_001` |
| `parent_checkpoint_id` | 当前快照从哪份快照继续产生 | `cp_001` 的后继可以把它记为父快照 |

**一个 thread_id 下可以有很多 checkpoint_id。checkpoint_id 不是 user_id，也不是包含多个会话的集合。**

业务上可以是“一个用户有多个会话，每个会话有多个状态快照”；但用户和会话的归属关系需要业务系统自己维护，不能把 Checkpointer 当作用户档案库。

### 2.1 用两轮聊天理解状态快照

假设一直使用 `thread_id = chat_001`：

```text
用户：你好
AI：你好呀

用户：你知道 LangGraph 吗？
AI：知道，它可以组织 Agent 的执行流程。
```

为了看清关系，只挑出四份与消息变化有关的快照：

| 快照 | checkpoint_id | messages 的内容 |
| --- | --- | --- |
| CP1 | cp_001 | [用户：你好] |
| CP2 | cp_002 | [用户：你好, AI：你好呀] |
| CP3 | cp_003 | [用户：你好, AI：你好呀, 用户：你知道 LangGraph 吗？] |
| CP4 | cp_004 | [用户：你好, AI：你好呀, 用户：你知道 LangGraph 吗？, AI：知道……] |

**CP 是 Checkpoint 的缩写，意思就是检查点／状态快照。** CP1、CP2 是讲解时起的名字，真实 ID 是可排序的 UUID。实际运行还会产生其他内部快照，不能按“一条消息必然对应一份快照”理解。

这里 `thread_id` 始终不变，`checkpoint_id` 随着新快照产生而变化。

### 2.2 parent_checkpoint_id 表达状态的来路

上述例子可以简化成：

| thread_id | checkpoint_id | parent_checkpoint_id |
| --- | --- | --- |
| chat_001 | cp_001 | NULL |
| chat_001 | cp_002 | cp_001 |
| chat_001 | cp_003 | cp_002 |
| chat_001 | cp_004 | cp_003 |

```text
CP1 → CP2 → CP3 → CP4
```

`CP4.parent_checkpoint_id = CP3` 表示 CP4 从 CP3 的状态继续产生。

它记录的是继承关系，也支持从历史状态产生分支：

```text
CP1 ── CP2 ── CP3 ── CP4
         │
         └──── CP5 ── CP6
```

此时 `CP5.parent_checkpoint_id = CP2`。有分支时，父快照不一定是按时间排序紧邻的上一条数据库记录。

对于本文的普通状态字段，恢复 CP4 可以根据 CP4 自己的字段版本引用读取数据，不需要沿着 CP1、CP2、CP3 逐个累加消息。后面提到的 Delta 持久化属于不同路径。

## 3. checkpoints 表：保存快照结构与字段版本引用

PostgreSQL 建表后的字段如下：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `thread_id` | TEXT | 会话／执行线程 ID |
| `checkpoint_ns` | TEXT | 命名空间，默认空字符串；可区分父图、子图等运行状态 |
| `checkpoint_id` | TEXT | 这份快照的 ID |
| `parent_checkpoint_id` | TEXT，可空 | 父快照 ID |
| `type` | TEXT，可空 | 表结构中保留的字段；当前常规快照写入 SQL 没有使用它 |
| `checkpoint` | JSONB | 快照结构：字段版本、部分字段值和执行状态信息等 |
| `metadata` | JSONB | 快照来源、执行步骤等元数据 |

联合主键：

```text
(thread_id, checkpoint_ns, checkpoint_id)
```

比如 CP2 的 `checkpoint` 中，下面这部分表示它引用的字段版本：

```json
{
  "id": "cp_002",
  "channel_versions": {
    "messages": "v2",
    "search_results": "v1"
  }
}
```

这是为了讲解而截取的结构，不是完整快照。

含义是：**恢复 CP2 时，messages 要取 v2，search_results 要取 v1。** 一份整体状态快照可以同时引用多个字段的不同版本。

## 4. checkpoint_blobs 表：保存字段的实际数据

### 4.1 channel 就是“这份数据属于哪个状态字段”

先把 Graph State 理解成一个字典：

```python
state = {
    "messages": [...],
    "search_results": [...],
    "current_task": "回答用户问题",
}
```

在常见的 StateGraph 中，这些字段分别对应内部 Channel：

| channel | 存什么 |
| --- | --- |
| `messages` | 消息列表 |
| `search_results` | 搜索结果 |
| `current_task` | 当前任务描述 |

Channel 还负责字段的更新、合并等行为，运行时也有内部调度用的 Channel。理解表结构时，先把它当作“字段名”就够了。

### 4.2 version 是某个字段的数据版本

假设原来：

```text
messages       → v1 → [用户：你好]
search_results → v1 → []
```

AI 回复后，可以简化理解为：

```text
messages       → v2 → [用户：你好, AI：你好呀]
search_results → v1 → []
```

消息字段更新了，搜索结果字段没有更新。新快照可以引用新的 messages 版本，同时复用原来的 search_results 版本。

**checkpoint_id 标识整个状态快照；version 标识其中某个 Channel 的版本，两者不是同一个东西。**

`v1`、`v2` 是教学简写。PostgreSQL Saver 实际生成的版本字符串包含递增数字前缀和随机后缀，应作为不透明标识使用；不要依赖它等于“第几条消息”。更严格地说，它追踪 Channel 的运行时更新版本，不是内容哈希，也不保证写入相同内容就不产生新版本。

### 4.3 type 与 blob

```text
type = msgpack
blob = 序列化后得到的字节数据
```

`type` 告诉恢复程序使用什么格式解码；`blob` 保存实际字节。PostgreSQL 使用 `BYTEA` 存这些字节。

不是把自然语言转成 Embedding。保存的是整个对象，例如消息列表中的角色、文本、消息 ID、工具调用及其他对象字段；反序列化后可以恢复这些结构。

默认的 `JsonPlusSerializer` 通常通过 `ormsgpack` 编码这类对象。虽然名字里有 JSON，但消息 Blob 通常不是直接可读的 JSON 文本；`type` 也不一定永远是 `msgpack`，例如原始 bytes 有自己的类型标记。

### 4.4 完整字段：共 6 个

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `thread_id` | TEXT | 属于哪个会话 |
| `checkpoint_ns` | TEXT | 属于哪个命名空间 |
| `channel` | TEXT | 哪个状态字段／内部 Channel |
| `version` | TEXT | 该 Channel 的版本 |
| `type` | TEXT | 编码类型，例如 msgpack |
| `blob` | BYTEA，可空 | 编码后的实际数据 |

联合主键：

```text
(thread_id, checkpoint_ns, channel, version)
```

教学例子中，表内的数据可以展示成：

| thread_id | checkpoint_ns | channel | version | type | blob（解码后） |
| --- | --- | --- | --- | --- | --- |
| chat_001 | 空字符串 | messages | v1 | msgpack | [用户：你好] |
| chat_001 | 空字符串 | messages | v2 | msgpack | [用户：你好, AI：你好呀] |

这张表没有 `checkpoint_id`。因为快照通过 **会话 + 命名空间 + Channel + version** 找到它引用的数据：

```text
checkpoints 中的 CP2
   └─ channel_versions.messages = v2
        ↓
checkpoint_blobs 中查找
   thread_id = chat_001
   checkpoint_ns = ""
   channel = messages
   version = v2
        ↓
反序列化，得到完整 messages 列表
```

还有一个存储例外：当前 PostgreSQL Saver 会把 `None`、字符串、整数、浮点数、布尔值直接保留在 `checkpoints.checkpoint.channel_values` 中；复杂值通常放入 Blobs。因此恢复状态需要结合快照内的值和 Blob，而不是所有数据都去 Blobs 查。

## 5. checkpoint_writes 表：保存尚未并入下一份快照的结果

### 5.1 为什么有了快照，还需要 Writes

LangGraph 会在多个执行阶段之间保存快照，**不是必须等“用户提问 → AI 最终回复”全部结束后才保存一次**。

先把一个 Superstep 理解为：这一阶段的节点读取当前状态并执行，之后统一合并它们产生的更新，推进到下一阶段。同一阶段可以有多个任务并行运行。

问题在于：阶段还没结束时，可能已有部分节点成功。如果此时其他节点失败，下一份完整快照尚未形成，已经成功的结果也需要保留下来。

Writes 就是用来保存这些节点中间结果的。正常执行也会写，不是只有异常时才写。

### 5.2 完整字段：共 9 个

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `thread_id` | TEXT | 会话 ID |
| `checkpoint_ns` | TEXT | 命名空间 |
| `checkpoint_id` | TEXT | 这次任务基于哪份快照执行 |
| `task_id` | TEXT | 这次任务执行的标识，不是固定的节点名称 |
| `task_path` | TEXT | 内部任务路径，用于执行定位与确定写入顺序等 |
| `idx` | INTEGER | 任务内写入序号；特殊写入可以使用负数 |
| `channel` | TEXT | 写入哪个字段／内部 Channel |
| `type` | TEXT，可空 | 编码类型 |
| `blob` | BYTEA | 本次写入的值，序列化后的字节 |

`task_path` 是后续迁移添加的字段。联合主键：

```text
(thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
```

一个任务可以写多个字段，因此可能对应多行记录。Writes 中也可以出现错误、中断等内部记录，不能把“存在任意一行”简单等同于“任务成功”。执行引擎还会判断任务状态。

### 5.3 串起一次最简单的聊天

继续前面的例子，当前是 CP1：

```text
CP1 的 messages = [用户：你好]
```

chatbot 节点执行后返回更新：

```python
{"messages": [AIMessage(content="你好呀")]}
```

节点返回的是本次更新，不必返回整个历史消息列表。

Writes 中可以新增这样一行：

| 字段 | 示例值 |
| --- | --- |
| thread_id | chat_001 |
| checkpoint_ns | 空字符串 |
| checkpoint_id | cp_001 |
| task_id | task_chatbot_001（示意 ID） |
| task_path | 内部执行路径，此处省略具体值 |
| idx | 0 |
| channel | messages |
| type | msgpack |
| blob（解码后） | [AI：你好呀] |

**这里挂的是 cp_001，因为任务是在 CP1 基础上执行的，不是挂尚未生成的 CP2。**

随后，这一阶段结束，消息归约函数把更新合并到旧状态：

```text
[用户：你好] + [AI：你好呀]
   ↓
[用户：你好, AI：你好呀]
```

产生 CP2，并保存新的消息版本 v2：

```text
读取 CP1
   ↓
执行 chatbot，得到 [AI：你好呀]
   ↓
保存节点写入
   └─ checkpoint_writes：CP1 / 本次 task / messages / [AI：你好呀]
   ↓
阶段结束，合并更新
   ↓
保存 CP2
   ├─ checkpoint_blobs：messages / v2 / [用户：你好, AI：你好呀]
   └─ checkpoints：CP2 / parent=CP1 / messages 引用 v2
```

这是常规阶段持久化路径的逻辑顺序，不意味着每次数据库调用都在主执行线程里同步等待；实际等待时机还受 durability 模式影响。

默认 `MessagesState` 用 `add_messages` 合并消息：新消息通常追加，有相同消息 ID 时可更新已有消息，也支持删除消息。所以它比普通 `list.append()` 更完整。

## 6. 并行任务失败时，Writes 怎样帮助恢复

假设从 CP10 同时执行两个任务：

```text
                    CP10
                      │
             ┌────────┴────────┐
             │                 │
          查询订单           查询物流
          Task A             Task B
             │                 │
           成功              超时失败
             │
             ↓
     保存 Task A 的结果到 Writes
```

Task A 返回：

```python
{"order_info": {"status": "paid"}}
```

这份结果已经成功持久化，但阶段没有完整结束，还没有包含两个任务结果的 CP11。

从失败点恢复时，可以：

1. 读取 CP10 的状态。
2. 读取 CP10 关联的 Pending Writes。
3. 根据任务标识与状态，把 Task A 的成功结果恢复到执行任务中。
4. 继续处理未完成的 Task B；按配置重新执行或走错误处理路径。
5. 阶段完成后合并更新，保存下一份快照。

**价值：阶段只完成一部分时，也能保留已经成功的部分，减少重复执行。**

这是从失败点继续的恢复语义。主动从历史检查点做 replay／time travel 可能重新执行后续节点，不能把所有“重新运行”都理解成复用成功结果。

## 7. 下一轮对话怎么恢复

两轮调用使用相同 `thread_id`，且对应命名空间一致：

```python
config = {"configurable": {"thread_id": "chat_001"}}
```

没有显式指定 `checkpoint_id` 时，PostgreSQL Saver 查找该 thread 和命名空间下 ID 排序最新的一份快照：

```sql
SELECT ...
FROM checkpoints
WHERE thread_id = %s
  AND checkpoint_ns = %s
ORDER BY checkpoint_id DESC
LIMIT 1;
```

它通过快照中的 `channel_versions` 匹配 Blob，并读取同一份快照的 Pending Writes，返回完整的 `CheckpointTuple`。如果指定 `checkpoint_id`，则按该 ID 读取历史快照。

于是普通下一轮输入可以在已有状态上合并：

```text
恢复 messages = [用户1, AI1]
   ↓
合并本轮输入 [用户2]
   ↓
messages = [用户1, AI1, 用户2]
   ↓
继续运行 Graph
```

失败恢复则需要按恢复协议继续原任务，通常通过 `graph.ainvoke(None, config)` 恢复；人工中断另有 `Command(resume=...)` 等路径。再次传入新用户输入和恢复上一条未完成任务，不是完全相同的操作。

## 8. 对讨论中理解的精确确认

原来的理解是：

> checkpoints 像索引，找到某个会话最新状态对应的版本；再通过 Blobs 恢复内容。Writes 保存中间过程，避免必须等最终回复后才落库，否则程序中途退出会无法恢复。

可以保留这个方向，并做三点修正：

1. **版本是一组字段版本。** 快照会记录 messages、search_results 等各自的版本，不是整个会话只有一个 Blob 版本。
2. **LangGraph 本来就会在多个执行阶段之间保存快照。** Writes 更细，补上“同一个阶段里部分任务已成功，但下一份快照还没有形成”的空隙。
3. **Writes 的主要目的是故障恢复，不是保证数据库原子性。** 它刻意保留部分成功结果；事务原子性讨论的是一组数据库操作要么全部成功，要么全部失败。这是两个层次的问题。

可以这样记：**Checkpoint 保存阶段状态，Blob 保存字段值，Writes 保存阶段内任务已产生的更新。**

## 9. 可靠性、存储成本与使用边界

### 9.1 是否已经落库，要看 durability

| 模式 | 持久化时机 | 对恢复的影响 |
| --- | --- | --- |
| `sync` | 下一阶段开始前等待快照持久化 | 阶段边界的持久化保障更强，增加等待 |
| `async` | 与下一阶段执行并行持久化，默认模式 | 进程突然崩溃时可能丢失最近尚未完成持久化的状态 |
| `exit` | Graph 退出时持久化 | 执行中硬崩溃时缺少本次运行的中间恢复点 |

节点返回结果、程序提交保存任务、数据库确认写入完成，是不同时间点。上面的 Task A 能被复用，前提是其成功结果确实已经持久化，并按对应恢复路径继续。

### 9.2 默认 messages 更新，通常保存完整列表

Writes 保存本次更新，例如 `[AI回复]`；普通 messages Channel 的新版本 Blob 通常保存更新后完整列表，例如 `[用户1, AI1, 用户2, AI2]`。

未更新的 Channel 可以复用旧版本，因此不需要每份快照都重存所有字段。但 messages 持续增长时，新 Blob 仍会包含历史消息，长期存储可能有较多重复数据。

源码也包含 DeltaChannel 的增量写入与周期性快照路径；它不等于默认 `MessagesState`，不能认为默认消息持久化已经自动变成逐条增量存储。这里仅记录区别，具体配置与恢复算法未在本次讨论中展开。

### 9.3 无法自动保证外部副作用只执行一次

```text
工具调用外部接口成功，例如发邮件
   ↓
还没写入成功结果，进程就崩溃
   ↓
恢复后发现任务未完成
   ↓
工具可能再次执行
```

Checkpointer 无法独自消除这个窗口。支付、发邮件、业务写入等工具，仍要用稳定的幂等键、业务去重或查询外部操作结果来处理重复执行。

### 9.4 对话展示、执行恢复、长期记忆的职责不同

| 层次 | 主要职责 |
| --- | --- |
| Conversation Store | 会话列表、消息分页、用户归属、消息状态、UI 展示 |
| Runtime Checkpointer | 保存状态、任务结果和恢复执行所需的信息 |
| 长期记忆系统 | 跨会话保存与召回用户事实、经验等 |
| SSE／事件重放 | 让客户端重新接收事件、恢复展示；是否能恢复执行取决于额外的服务端协议 |

它们可以共享数据库或协作，但 Checkpointer 不会自动提供完整的用户档案、聊天产品查询模型或跨会话长期记忆。

## 10. 源码定位与阅读状态

以下链接固定到整理时核对的提交，避免以后 main 改动导致证据漂移。

| 已核对的内容 | 官方源码 |
| --- | --- |
| 三张表、联合主键、task_path 迁移、读取关联和写入 SQL | [postgres/base.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/checkpoint-postgres/langgraph/checkpoint/postgres/base.py#L46) |
| 按 ID 或最新快照读取、aput 保存、aput_writes 保存 | [postgres/aio.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py#L181) |
| Writes 序列化、特殊序号和 Channel 版本生成 | [postgres/base.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/checkpoint-postgres/langgraph/checkpoint/postgres/base.py#L589) |
| MessagesState 与消息合并规则 | [graph/message.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/langgraph/langgraph/graph/message.py#L64) |
| 类型标记、序列化和反序列化 | [serde/jsonplus.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/checkpoint/langgraph/checkpoint/serde/jsonplus.py#L280) |
| 提交 Pending Writes、阶段结束保存、恢复成功任务输出 | [pregel/_loop.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/langgraph/langgraph/pregel/_loop.py#L452) |
| durability 的默认值和三种模式 | [pregel/main.py](https://github.com/langchain-ai/langgraph/blob/a6731f436700c7d46a9224ccd3a0e0ce3a5bfdb5/libs/langgraph/langgraph/pregel/main.py#L2798) |

本次已讨论并写入：三张表的分工、字段与关联，Checkpoint 和 Channel version 的区别，正常保存／恢复过程，Pending Writes 与原子性的区别，副作用和存储成本边界。

尚未共同逐项拆解：Pregel 完整调用链与调度版本判断、任务标识生成、具体事务与连接行为、DeltaChannel 算法、人工中断与历史重放细节、数据库实际运行演示。本笔记不把这些内容标记为完整掌握。

## 11. 相关笔记

- [[agent/architecture-components/07-session-state-and-persistence/_component|会话状态与持久化：组件范围]]
- [[agent/architecture-components/07-session-state-and-persistence/hermes|Hermes：会话状态与持久化]]
- [[all_harness/designing-multiagent-systems/04_checkpoint_replay_and_idempotency|Checkpoint、重放、幂等与副作用]]
- [[all_harness/00_拆解共识|Harness 拆解共识]]
