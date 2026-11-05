# Agent 组件拆解框架

这个目录不是某一个 agent 框架的笔记，而是一套通用的“组件优先”解剖框架。

目标：

1. 先固定一套相对稳定的 agent 组件边界。
2. 后续分析 Hermes、OpenClaw、OpenHands、Claude Code、Codex 之类时，按组件落位。
3. 每个组件目录下可以继续放不同框架的对应笔记，而不是每次重新发明目录结构。

## 使用方式

推荐采用：

- 一个组件对应一个目录
- 每个组件目录先看 `_component.md`
- 后续每分析一个框架，就在该组件目录下新增对应笔记

例如：

- `03-memory-and-recall/hermes.md`
- `03-memory-and-recall/openclaw.md`
- `04-tool-runtime-and-execution/hermes.md`
- `04-tool-runtime-and-execution/openclaw.md`

## 这套组件边界的设计原则

这不是按“功能宣传词”分，而是按系统里真正独立演化、独立设计的能力层分。

比如：

- “记忆”单独成层，因为它通常涉及存储、召回、抽取、注入四个子问题
- “工具运行时”单独成层，因为它通常涉及 schema、调度、执行、回传、权限
- “session state”单独成层，因为它经常和 memory 混淆，但其实是另一套持久化逻辑

## 当前组件列表

1. [00-system-overview](00-system-overview/_component.md)
2. [01-agent-loop-and-turn-runtime](01-agent-loop-and-turn-runtime/_component.md)
3. [02-context-and-prompt-assembly](02-context-and-prompt-assembly/_component.md)
4. [03-memory-and-recall](03-memory-and-recall/_component.md)
5. [04-tool-runtime-and-execution](04-tool-runtime-and-execution/_component.md)
6. [05-planning-and-task-decomposition](05-planning-and-task-decomposition/_component.md)
7. [06-subagents-and-multi-agent-coordination](06-subagents-and-multi-agent-coordination/_component.md)
8. [07-session-state-and-persistence](07-session-state-and-persistence/_component.md)
9. [08-model-routing-and-provider-abstraction](08-model-routing-and-provider-abstraction/_component.md)
10. [09-interface-and-channel-adapters](09-interface-and-channel-adapters/_component.md)
11. [10-plugin-and-extension-system](10-plugin-and-extension-system/_component.md)
12. [11-safety-guardrails-and-approval](11-safety-guardrails-and-approval/_component.md)
13. [12-observability-and-debugging](12-observability-and-debugging/_component.md)

## 推荐的分析顺序

如果第一次拆一个 agent，建议顺序：

1. `00-system-overview`
2. `01-agent-loop-and-turn-runtime`
3. `04-tool-runtime-and-execution`
4. `03-memory-and-recall`
5. `07-session-state-and-persistence`
6. `02-context-and-prompt-assembly`
7. 其他扩展层

原因：

- 先把主循环跑通
- 再看工具怎么接进来
- 再看 memory / session / prompt 怎么围绕主循环建立状态

## 与现有笔记的关系

当前 `hermes/` 目录仍然保留，适合做“按框架视角的连续解剖”。

这个 `architecture-components/` 目录是另一种视角：

- `hermes/`：按框架串讲
- `architecture-components/`：按组件对比

两套视角可以并存，不需要马上迁移旧笔记。
