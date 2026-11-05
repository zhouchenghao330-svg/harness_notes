# 01 Built-in Memory

## 关注点

- `MEMORY.md` / `USER.md` 的职责划分
- 写入接口
- 注入时机
- 为什么是 frozen snapshot
- 写入来源到底有几条

## 已确认

- Hermes 内置记忆使用 `~/.hermes/memories/MEMORY.md` 和 `~/.hermes/memories/USER.md`。
- 这两份文件在 session 启动时从磁盘加载。
- 注入到 system prompt 的是 frozen snapshot，而不是 live state。
- 运行中通过 `memory` 工具写入时，会立刻落盘，但不会热更新当前 session 的 system prompt。

## 一条存储链，两种写入来源

先把结构讲清楚：

- Built-in memory 的底层存储只有一套。
- 最终写入入口只有一个：`memory` tool 对应的 `MemoryStore`。
- 但触发“要不要写”的上层决策者有两种。

### 1. 前台主 agent 即时写入

主 agent 在正常对话过程中，如果意识到某条信息值得长期保存，就会直接调用 `memory` tool。

这类写入的特点：

- 和当前任务强相关
- 发生在前台执行路径里
- 更像“当场意识到这条该记住”

### 2. 后台 review agent 补写

Hermes 还会在一定 turn cadence 下触发 background review。这个 reviewer 会回看刚才的对话快照，再判断是否有值得沉淀进 built-in memory 的内容。

这类写入的特点：

- 发生在主对话结束之后
- 属于补采样、补整理
- 更像“前台没记，后台再回顾一遍看看有没有漏掉的 durable facts”

### 关键边界

不要把它理解成两套 memory 系统。

更准确的说法是：

- 同一个 built-in memory 存储层
- 两种写入触发机制
- 最终都落到同一个 `MEMORY.md / USER.md` 持久化链路

## 触发层和存储层要分开看

这一点很重要。

Hermes 的 built-in memory 至少要拆成两层：

### A. 存储层

- `MEMORY.md`
- `USER.md`
- `MemoryStore`
- 文件锁、drift 检测、落盘、容量限制、snapshot 冻结

这一层偏工程实现。

### B. 写入决策层

- 主 agent 在对话中主动决定写入
- reviewer 在后台复盘后决定补写

这一层偏 agent 行为。

所以如果只看 `memory_tool.py`，会以为 built-in memory 只是本地文件 CRUD；但真正让它变成“会积累”的，是上面的写入决策层。

## 暂时不展开的部分

当前先只确认“有哪些写入来源”。

先不展开：

- 主 agent 在不同业务场景下为何会决定写 memory
- reviewer 的具体提示词如何引导记忆抽取
- 不同场景下 memory 写入偏向 `memory` 还是 `user`

这些属于“写入提示词与策略”层，后面单独拆。

## 这层的意义

- 让最关键的 durable facts 常驻上下文。
- 控制 prompt 体积，避免长期记忆无限膨胀。
- 保持 prefix cache 稳定，不因为中途写 memory 导致每轮 system prompt 漂移。

## 还要继续看的点

- `memory` 工具的 `add / replace / remove` 调用分布
- 容量限制达到后，agent 的整理策略
- threat scanning 对 memory 内容的影响
- 主 agent 与 reviewer 的写入边界
- reviewer 的 cadence 和触发条件

## 代码入口

- `tools/memory_tool.py`
- `agent/agent_init.py`
- `agent/system_prompt.py`
- `agent/conversation_loop.py`
- `agent/background_review.py`
