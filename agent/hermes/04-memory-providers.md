# 04 Memory Providers

## 关注点

- 外部 provider 在 Hermes 里是替代 built-in，还是叠加 built-in
- provider 是本地能力，还是外部服务
- 不同 provider 代表什么架构方向

## 已确认

- Hermes 只允许同时启用一个 external memory provider。
- external provider 是 additive，不会替换 built-in memory。
- provider 的初始化、prefetch、sync、tool surface 都通过 `MemoryManager` 统一编排。

## 两条典型路线

### 1. Service-backed provider

代表：

- Honcho
- Mem0
- Supermemory

特点：

- 用户建模和长期语义记忆能力更强
- 可能依赖云服务或自托管后端
- “内部怎么总结 / 怎么建模”不一定完全透明

### 2. Local provider

代表：

- Holographic

特点：

- 本地 SQLite / FTS5 / trust scoring
- 设计更可拆、更适合单机研究
- 更像可控的 memory substrate，而不是远端 persona engine

## 当前判断

- 如果目标是研究“默认 Hermes 怎么做长期记忆”，重点应先放在 built-in memory + session recall + background review。
- 如果目标是研究“更强的用户模型怎么做”，再进入 Honcho 这类 provider。

## 还要继续看的点

- `MemoryProvider` ABC 的最小语义边界
- `prefetch` 和 `sync_turn` 的调用时机
- Honcho 的 session summary / representation / dialectic 是怎么拼装的
- Holographic 的 local fact graph 是否值得单独对比 OpenClaw / OpenHands 一类实现

## 代码入口

- `agent/memory_provider.py`
- `agent/memory_manager.py`
- `plugins/memory/honcho/`
- `plugins/memory/holographic/`
