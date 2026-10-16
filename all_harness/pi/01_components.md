# Pi Harness 组件索引

> Pi 是第一个拆解样本。每个组件先共同阅读，再决定写成单个 Markdown 文件，还是拆成一个文件夹。

## 组件组织原则

组件的边界按 Agent 行为判断，不按目录名判断。

如果一个模块会改变以下任一项，就属于 Harness 组件范围：

- 模型输入或输出
- Agent Loop
- Context、Prompt 或 Tool Definitions
- Tool 执行和结果
- State、History、Memory、Compaction
- Continue、Stop、Retry、Cancel、Approval
- Session 持久化、恢复或分支
- Agent 事件、流式输出或 Extension 行为

组件笔记必须回答：

1. 它解决什么问题？
2. 什么时候被触发？
3. 输入和输出是什么？
4. 状态存在哪里？
5. 它和哪些组件连接？
6. 它如何影响 Agent 下一步？
7. 出错、重试、取消时怎么处理？
8. 当前设计的优点、代价和限制是什么？

## Pi 组件地图

| 编号 | 组件 | 主要代码入口 | 当前状态 |
| --- | --- | --- | --- |
| 01 | AgentSession Orchestrator | coding-agent/src/core/agent-session.ts | 待共同阅读 |
| 02 | Agent Loop | agent/src/agent-loop.ts | 已共同阅读 |
| 03 | Agent State and Lifecycle | agent/src/agent.ts | 已共同阅读 |
| 04 | Context and Prompt | coding-agent/src/core/system-prompt.ts、resource-loader.ts、agent-loop.ts | 已共同阅读 |
| 05 | Tool System and Concrete Tools | coding-agent/src/core/tools、agent/src/agent-loop.ts | 部分完成：框架、read、write、edit、bash 已共同阅读 |
| 06 | Session and Persistence | coding-agent/src/core/session-manager.ts | 已共同阅读 |
| 07 | Compaction and Branching | coding-agent/src/core/compaction、agent-session.ts | 已共同阅读 |
| 08 | Model and Provider Adapter | coding-agent/src/core/model-runtime.ts、packages/ai | 已共同阅读 |
| 09 | Extensions, Skills and Resources | coding-agent/src/core/extensions、skills.ts、resource-loader.ts | 已共同阅读 |
| 10 | Control, Queues and Cancellation | agent/src/agent.ts、agent-loop.ts、agent-session.ts | 已共同阅读 |
| 11 | Events, Streaming and Output | agent/src/types.ts、agent.ts、coding-agent/src/modes | 已共同阅读 |

## 单个组件的推荐结构

### 简单组件

直接使用一个 Markdown 文件：

- problem.md
- architecture.md
- code_path.md
- design_tradeoffs.md

### 复杂组件

使用一个文件夹：

components/<component-name>/
  ├── 00_overview.md
  ├── 01_architecture.md
  ├── 02_runtime_flow.md
  ├── 03_code_evidence.md
  └── 04_design_tradeoffs.md

## 研究顺序

第一阶段只完成：

1. Agent Loop
2. Agent State and Lifecycle
3. Context and Prompt
4. Tool System
5. Session and Persistence

之后再看：

6. Compaction and Branching
7. Model and Provider Adapter
8. Extensions and Resources
9. Control and Output

## 研究纪律

- 先在对话中共同阅读和确认，再写组件细节。
- 组件索引可以先建立，但不能把未确认的推断写成事实。
- 每条关键结论绑定文件、函数、类型或调用链。
- 区分 Code-verified、Docs-described 和 Inferred。
- 不把 TUI 或 CLI 外壳自动当成 Agent Core。
- 不把业务目录自动排除；如果它改变 Agent 行为，就纳入。
- 当前阶段不修改源码。
