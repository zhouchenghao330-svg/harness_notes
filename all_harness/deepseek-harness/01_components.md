# DeepSeek Harness 组件索引

> 状态：研究框架已建立，已确认的基础概念见 [[00_cordis_and_tool_basics]]，应用与 Agent 装配的职责边界见 [[00_profile_bundle_preset]]。其余操作保留各自阅读状态，主链预读不等于已经讲解或完成。
>
> 总览：[[00_overview]]；总规则：[[all_harness/00_拆解共识]]。

## 1. 范围与阅读状态

按 Agent 行为划分组件，不把每个 package 当成一个待讲模块。同一个行为可能横跨多个插件；同一个插件也可能服务于不同研究问题。DSH 的特有机制保留独立位置，不强行套用 Pi 或 Hermes 的内部结构。

阅读状态使用：待共同阅读、部分阅读、已共同阅读、用户明确暂缓、待验证。源码预读程度和笔记写入状态分别记录。已写入总览、索引及限定范围的基础概念笔记；各组件的完整操作仍需逐项共同阅读。

## 2. 组件地图

下表中的入口是研究起点，不是穷尽后的完整调用链。编号沿用已有笔记的主要主题，前置知识和入口单独列出。

| 编号 | 组件 | 主要源码入口 | 要回答的行为问题 | 状态 |
| --- | --- | --- | --- | --- |
| 前置 | Cordis 与运行组合 | `apps/cli/src/profile-boot.ts`、`packages/bundle/base/cordis.patch.yml`、`packages/boot/app-boot/src/` | 插件怎样提供能力、互相连接、按配置启动并在卸载时清理？ | 部分阅读：Cordis 定位及 Profile/Bundle/Preset 职责已确认；事件、依赖生命周期、Patch 与具体隔离机制待共同阅读 |
| 入口 | 各运行外壳与任务提交 | `apps/cli/src/bin.ts`、`packages/bundle/headless/src/index.ts`、`packages/api/`、`packages/sdk/server/src/index.ts`、`packages/acp/acp/src/index.ts` | 一次用户输入怎样选中会话并进入 Agent？不同入口有什么限制？ | 待共同阅读 |
| 02 | Agent Loop | `packages/core/agent-loop/src/agent.ts` | 模型、工具、消息和停止判断怎样构成一次运行？ | 待共同阅读 |
| 03 | Agent State / Lifecycle / Inbox | `packages/core/agent/src/`、`packages/core/agent-loop/src/index.ts`、`packages/core/agent-loop/src/inbox.ts` | 谁创建、持有和销毁 Agent？输入、控制信号和活动状态存在哪里？ | 待共同阅读 |
| 04 | Context / Prompt / Instructions | `packages/core/system-prompt/src/index.ts`、`packages/core/agent-loop/src/runtime-context.ts`、`packages/context/` | 模型究竟看到哪些提示、指令、历史和动态上下文？ | 待共同阅读 |
| 05 | Tool System / Native / PTC / Concrete Tools | `packages/core/tools/src/`、`packages/core/tool-executor-native/src/`、`packages/core/tool-executor-ptc/src/`、`packages/ptc-runtime/` | 工具怎样注册、暴露、执行、返回；原生调用与程序调用有什么差别？ | 部分阅读：仅说明、调用与底层实现的概念分工；工具体系、清单与具体工具待共同阅读 |
| 06 | Model / Provider / Request / Streaming | `packages/llm/llm/src/`、`packages/llm/llm-deepseek/`、`packages/llm/llm-pi-ai/` | 内部请求怎样适配到模型协议，再还原文本、推理、工具调用和错误？ | 待共同阅读 |
| 07 | Session / Persistence / Projection / Recovery | `packages/core/session/src/`、`packages/session/session-persistence*/`、`packages/session/session-projection/`、`packages/session-query/` | 事件怎样变成模型历史、派生状态、持久记录和可恢复会话？ | 待共同阅读 |
| 08 | Compaction / Context Retention / Memory | `packages/compaction/`、`packages/core/session/src/surface.ts` | 上下文怎样缩减、替换和恢复？是否还有独立长期记忆？ | 待共同阅读；长期记忆待验证 |
| 09 | Control / Retry / Cancel / Approval / Sandbox | `packages/llm/llm-retry/`、`packages/interaction/`、`packages/guard/`、`packages/sandbox/` | 运行何时重试、取消、等待人工、被拒绝或超时？ | 待共同阅读 |
| 10 | Delegation / Subagents | `packages/subagent/subagent/src/`、`packages/subagent/subagent-spawn-in-process/`、`packages/subagent/subagent-fork-in-process/`、`packages/subagent/tool-subagent/` | 子任务怎样建立、隔离、继续、交互和回传结果？ | 待共同阅读 |
| 11 | Events / Live Output / Transcript / Deliverables | `packages/core/agent-loop/src/assistant-stream.ts`、`packages/core/agent/src/types.ts`、`packages/bundle/headless/src/json-stream.ts`、`packages/client/`、`packages/deliverables/` | 用户看到的实时输出、最终回答和交付物怎样与 Session 对齐？ | 待共同阅读 |
| 12 | Skills / MCP / Runtime Extensions | `packages/skill/`、`packages/mcp/`、`packages/extensions/`、`packages/boot/plugin-manager/` | 新能力怎样发现、加载、注入提示、注册工具并被卸载？ | 待共同阅读 |
| 13 | Capability Providers / Execution Environment | `packages/fs/`、`packages/shell/`、`packages/subprocess/`、`packages/terminal/`、`packages/ssh/`、`packages/browser-use/`、`packages/computer-use/` | 同一工具如何在不同文件、进程、远程或浏览器环境执行？ | 待共同阅读 |
| 14 | Presets / Per-agent Composition | `packages/core/scope/`、`packages/preset/agent-preset/`、`packages/preset/agent-preset-registry/src/`、`packages/preset/persona/` | 不同 Agent 怎样获得自己的提示、工具、服务和配置，并在恢复后延续选择？ | 部分阅读：职责、装配关系、复用与修订边界已记录；同名贡献选择、隔离、失败及生命周期完整路径待共同阅读 |
| 15 | Goals / Plan / Todo / Workflow / Schedule / Jobs | `packages/goal/`、`packages/plan/`、`packages/todo/`、`packages/workflow/`、`packages/schedule/`、`packages/jobs/` | 目标、计划、后台任务、工作流和定时输入怎样改变后续运行？ | 待共同阅读 |
| 16 | Attachments / Spill / References | `packages/attachment/`、`packages/spill/`、`packages/context/file-reference/`、`packages/context/session-reference/`、`packages/session/working-directory/` | 图片、大结果、文件引用和工作目录怎样影响模型输入及工具后续动作？ | 待共同阅读 |

实验能力另行核对：`packages/experimental/agent-team/`、`packages/experimental/webhook/` 以及实验性浏览器、计算机和 PTC provider。它们的入口已定位，尚未分析实际配置和调用链；确认后归入对应组件，不能默认排除，也不视为基础运行路径已启用。

## 3. 各组件的初始操作清单

以下是后续必须检查的问题，不是已经确认的实现结论。沿实际调用链继续补齐；没有讲解的项目保持待共同阅读。

### 前置与入口

- 服务、事件、可释放注册、依赖激活和插件卸载的最小概念。
- profile、bundle、用户 patch 和命令行 patch 怎样决定实际插件组合。
- headless 的新建与恢复、模型选择、权限设置、任务提交、等待子任务、flush 和退出。
- Web、Desktop、SDK、ACP 的输入入口、会话归属、恢复与运行控制；对比 headless 的差异。

### 02：Agent Loop

- turn、step、物理模型请求之间的关系；首次输入为空或被拒绝时的行为。
- 领取消息、选择工具、组装上下文、接纳输入、准备适配器、记录请求和发送请求的先后顺序。
- 文本、工具调用、长度上限、失败响应、中断响应的分支。
- 工具结果怎样回到后续请求；工具批次怎样要求结束或附加上下文。
- 重试是否重做组装和输入接纳；停止前扩展点怎样让任务继续。
- 每条失败路径如何补齐缺失的工具结果、关闭 step 和 turn。

### 03：状态、生命周期与输入队列

- Agent、Session、注册表、作用范围及持久化句柄的持有关系。
- 新建、恢复、发布、初始化失败回滚和销毁顺序。
- `followup`、`steer`、`inject` 的目标队列、唤醒规则和消息领取时机。
- 空闲、运行、维护任务之间的切换；取消和退出期间新输入怎样处理。
- `whenIdle()` 等待的具体范围与子任务等待的区别。

### 04：上下文、提示与指令

- 提示段、动态上下文、工具声明、变量与完整提示覆盖的组装规则。
- AGENTS.md 等指令怎样发现、读取、更新并进入模型上下文。
- system 消息与动态 user-role 上下文的区别、刷新时机和日志记录。
- 文件、会话、时间、工作目录等上下文来源，以及不同模型能力对提示更新的影响。
- 最终请求 payload、请求序列和压缩后的上下文恢复。

### 05：工具体系与具体工具

- 注册、作用范围覆盖、限制、条件启用、每一步的可用工具快照与释放。
- 输入 schema、输出 schema、结构化返回值、模型可见 content 与展示元数据。
- 批处理并发、顺序约束、参数校验、审批、guard、执行、后处理、超时、取消和失败。
- 原生执行器如何直接暴露工具；PTC 如何暴露 `run_code`、生成 SDK、执行程序及其嵌套工具调用。
- 程序运行环境、语言 provider、上下文隔离、输出控制和会话记录。
- 建立 `05_tools/00_inventory.md`：记录准确工具名、来源插件、配置启用条件、模型可见形式、主要副作用和共同阅读状态。
- 再按文件、搜索、Shell/进程、网络、浏览器/计算机、人工交互、子 Agent、目标/计划、技能/MCP、扩展和交付等能力逐个或分组拆解。

当前尚未建立完整工具清单。共享 base 的插件行只是初始线索，不能代替工具定义核对；工具章节完成需要框架、清单、核心工具阅读三部分都完成。

### 06：模型与协议适配

- 路由注册、模型选择、能力发现、凭据、默认参数和参数校验。
- 准备调用时怎样固定适配器及其能力，何时才能提交模型可见输入。
- 请求转换、工具历史、推理内容、多模态输入、响应组装与流结束原因。
- 错误分类与重试交接；模型流、普通失败和取消的区别。
- DeepSeek 与其他 provider 的实际请求，不能只读公共接口。

### 07：会话、持久化与恢复

- 追加日志、当前生效消息、请求配置、消息投影和派生状态的关系。
- 内存追加、检查点、flush、后台写入和 close 的保证及失败语义。
- JSONL 存储、写入占用、压缩文件、格式代际和相邻版本迁移。
- 恢复时未完成 turn、step、工具调用怎样补齐；fork、继承和分支怎样表示。
- 投影注册与增量折叠、投影缓存、会话查询、搜索和导出。

### 08：压缩、上下文保留与记忆

- 自动压力触发、切换模型、请求超限、人工压缩及各自失败策略。
- 保留区间、摘要模型与提示、工具配对、压缩互斥和取消。
- 工具结果裁剪、图片卸载、当前消息节点替换、请求序列变化和恢复后的行为。
- 分别确认历史、会话检索、压缩摘要、外部记忆工具的职责；独立长期记忆目前待验证。

### 09：运行控制

- 用户取消、生命周期销毁、超时、长度上限、模型失败和工具失败分别怎样收束。
- provider 重试策略、等待与退避、持久化重试状态和重启后的计数。
- 权限预设、人工审批、运行 guard、文件/网络/进程沙箱怎样分工。
- 拒绝或中断后的模型反馈，以及输入队列保留或清理的规则。

### 10：委派与子 Agent

- provider 能力、spawn 与 fork、上下文和 preset 选择、父子身份与深度。
- 前台和后台执行、继续已有子任务、消息传递、取消、结果投递与资源回收。
- 本进程子 Agent 与外部 SDK / ACP / 其他 Agent provider 的差异。
- 子任务发现、持久化目录、恢复、父任务空闲后的唤醒和结构化输出。
- 实验性 Agent Teams 的任务依赖、共享状态和交互方式另行核对。

### 11：事件、输出与交付

- Session 持久事件、Agent 实时事件和能力事件分别供谁消费。
- 响应流开始、增量、结束与完整回答或失败尝试记录的关系。
- headless、Web、SDK、ACP 的输出差异；断连、恢复与中断输出。
- 工具展示、最终回答、文件交付及工作区变更怎样从记录生成。

### 12：技能、MCP 与动态扩展

- 技能来源、发现、选择、读取、指令注入、作用范围和更新。
- MCP 连接、工具与资源转换、名称和 schema、错误、断线与卸载。
- 运行时加载 Host / Client 插件、安装管理与能力变化怎样影响当前 Agent。
- 这些扩展对提示、工具、日志、权限和恢复的影响。

### 13：能力提供者与执行环境

- 以文件读取、命令执行各走一遍：工具消费者、服务定义、provider 的完整链。
- 本地、沙箱和远程 provider 的选择怎样影响路径、文件、进程及网络。
- 临时命令、持续终端、后台 job、子进程的状态与清理。
- 浏览器、计算机和 SSH 能力的实际组成、权限、取消与失败。

### 14：Preset 与单 Agent 组合

- 全局贡献和当前 Agent 内贡献的继承、覆盖、限制与资源释放。
- preset 定义、装载、选择、切换和持久化选择的恢复。
- persona、工具执行方式、模型和子 Agent 组合之间的关系。
- 配置隔离与执行环境隔离分别提供什么保证。

### 15：目标、计划、工作流与调度

- goal、plan、todo 各自保存什么，哪些信息影响模型、工具或继续运行。
- 目标自动续跑、预算、停止条件、检查点、用户新输入和错误竞争。
- workflow 的执行依赖、子 Agent/PTC 配合与结果回传。
- 定时任务怎样唤醒会话；后台 job 怎样等待、查询、停止和恢复。

### 16：附件、引用与大结果

- 图片等二进制附件怎样保存、引用、读取并适配模型。
- 工具结果怎样裁剪、外溢、保存以及被后续工具取回。
- 文件引用、会话引用与工作目录解析怎样改变输入和执行路径。
- 权限、过期引用、缺失文件、取消和恢复时的处理。

## 4. 共同阅读顺序与写入计划

第一阶段：最小 Cordis 前置知识 → headless 一次任务 → 02 循环 → 03 状态与生命周期 → 04 上下文 → 05 工具框架与核心工具 → 07 Session。

第二阶段：06 模型适配 → 08 压缩 → 09 控制 → 10 子 Agent → 11 输出。读取前面组件时遇到这些机制，先解释它对当前步骤的作用，并保留对应组件的完整阅读任务。

第三阶段：12 扩展 → 13 能力提供者 → 14 preset → 15 长任务与调度 → 16 附件与引用；同时补齐其他入口、实验能力和具体工具。

后续详细笔记沿编号命名，如 `02_agent_loop.md`、`03_agent_state_and_lifecycle.md`、`04_context_and_prompt.md`、`05_tool_system.md`、`05_tools/00_inventory.md`。实际内容复杂时再拆子目录，不预建空文件。

## 5. 组件结束前的核对

- 操作清单是否已经沿调用链补齐，所有条件分支和后台动作是否有记录？
- 每项是否已经共同讲解；暂缓、待验证和未讲项是否仍明确保留？
- 默认配置、其他 profile、可选插件和实验能力是否区分清楚？
- 关键结论是否有源码证据，文档描述和推断是否单独标明？
- 工具组件是否同时覆盖体系、清单和核心工具？
- 笔记是否只写入已经讨论并获准落笔的内容，索引进度是否同步？

存在未讲或待验证的 Runtime 操作时，只能标记部分完成。用户确认当前一段已懂，不代表整个组件已经完成。
