# 03 Background Review

## 关注点

- Hermes 怎么把“会话里值得长期保留的内容”筛出来
- 这个筛选是规则驱动还是 LLM 驱动
- 触发条件是什么

## 已确认

- Hermes 有一个 background review 机制。
- 主对话结束后，会 fork 一个 review agent 去回看当前 conversation snapshot。
- review agent 只允许使用有限的工具面，重点包括 `memory` 和 `skill_manage`。
- memory review 不是单纯记录，而是要判断“值不值得写入长期记忆”。

## 这层的本质

- 存储层仍然可以是本地。
- 但“抽取哪些内容进入长期记忆”这一步已经是模型参与的语义筛选。

## 这层的重要性

- 这是 Hermes 从“手工记笔记系统”升级为“主动积累系统”的关键。
- 如果只看 `MEMORY.md / USER.md`，会误以为 Hermes 的长期记忆只是文件 CRUD。
- 真正有 agent 味道的部分，恰恰在这层。

## 还要继续看的点

- `nudge_interval` 的默认值和真实触发频率
- review prompt 的边界：偏用户画像，还是也写环境/workflow facts
- review 失败时有没有重试或降级

## 代码入口

- `agent/conversation_loop.py`
- `agent/background_review.py`
