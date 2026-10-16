# Pi Harness 拆解总览

> 本文记录 Pi 的第一轮 Agent-centric 扫描结果。它是架构地图，不是组件细节结论。后续组件要在共同阅读代码后分别补充。

## 1. 研究定位

Pi 是一个最小化的终端 coding harness。

研究目标不是解释整个 pi-mono 产品，而是拆出它如何把 LLM 变成一个可执行、可持续、可恢复的 Agent Runtime。

当前研究只关注：

- Agent Run 的生命周期
- Agent Loop
- State、History、Context
- Prompt 组装
- Tool 注册和执行
- Session 持久化、分支和 Compaction
- Model / Provider 适配
- Extension、Skill 和人工控制对 Agent 行为的影响

## 2. 当前主运行路径

当前默认可运行的主路径是：

```
CLI / Interactive / Print / RPC / SDK
  ↓
createAgentSession()
  ↓
AgentSession
  ↓
Agent
  ↓
runAgentLoop()
  ↓
packages/ai
```

关键代码：

- packages/coding-agent/src/main.ts：CLI 入口和模式选择
- packages/coding-agent/src/core/sdk.ts：创建 AgentSession
- packages/coding-agent/src/core/agent-session.ts：产品化 Agent 控制层
- packages/agent/src/agent.ts：有状态 Agent 外壳
- packages/agent/src/agent-loop.ts：通用执行循环
- packages/coding-agent/src/core/model-runtime.ts：模型和 Provider 运行时
- packages/coding-agent/src/core/session-manager.ts：会话持久化和上下文恢复

## 3. Agent-centric 架构图

```
┌────────────────────────────────────────────┐
│              外部运行外壳                  │
│ CLI / TUI / Print / RPC / SDK              │
└──────────────────┬─────────────────────────┘
                   ↓
┌────────────────────────────────────────────┐
│              createAgentSession             │
│                                            │
│ ModelRuntime                               │
│ ResourceLoader                             │
│ SessionManager                             │
│ SettingsManager                            │
└──────────────────┬─────────────────────────┘
                   ↓
┌────────────────────────────────────────────┐
│              AgentSession                   │
│                                            │
│ 输入预处理、扩展、Skill、Prompt Template    │
│ Tool Registry、System Prompt               │
│ Retry、Compaction、Branch                  │
│ Agent 事件监听和 Session 持久化             │
└──────────────────┬─────────────────────────┘
                   │ agent.prompt()
                   ↓
┌────────────────────────────────────────────┐
│                 Agent                      │
│                                            │
│ State：systemPrompt / model / tools        │
│        messages / streamingMessage          │
│ Steering Queue / Follow-up Queue            │
│ AbortController / Event Subscribers         │
└──────────────────┬─────────────────────────┘
                   │ runAgentLoop()
                   ↓
┌────────────────────────────────────────────┐
│              Agent Loop                    │
│                                            │
│ 组装当前 Context → 调用模型                 │
│ 判断 Tool Call → 执行工具                  │
│ 写入 Tool Result → 继续或停止              │
└───────────────┬────────────────────────────┘
                │
       ┌────────┴────────┐
       ↓                 ↓
  Model Adapter       Tool System
       │                 │
       ↓                 ↓
  packages/ai       参数校验 / Hook
                    执行 / 结果处理
                         │
                         ↓
                   Tool Result
                         │
                         └── 回到 Agent Loop
```

横切能力：

- SessionManager：保存消息、状态、分支和 Compaction
- ResourceLoader：加载 AGENTS.md、Skills、Prompt 和 Extensions
- ExtensionRunner：注入输入、Prompt、Tool Hook 和事件逻辑
- AgentSession：处理 Retry、Overflow Recovery 和 Auto-Compaction

## 4. 一次请求的调用链

```
用户输入
  ↓
AgentSession.prompt()
  ↓
扩展命令判断
  ↓
Input Hook
  ↓
Skill / Prompt Template 展开
  ↓
检查模型和认证
  ↓
检查是否需要 Compaction
  ↓
创建 user message
  ↓
执行 before_agent_start
  ↓
agent.prompt()
  ↓
runAgentLoop()
  ↓
streamAssistantResponse()
  ↓
transformContext()
  ↓
convertToLlm()
  ↓
ModelRuntime.streamSimple()
  ↓
模型返回文本或 Tool Call
```

Tool Call 分支：

```
Tool Call
  ↓
查找工具
  ↓
参数预处理和校验
  ↓
beforeToolCall
  ↓
执行工具
  ↓
afterToolCall
  ↓
生成 toolResult message
  ↓
写回 Context
  ↓
下一次模型调用
```

## 5. 第一轮判断

### 5.1 Agent Loop 很小

真正的循环主要集中在：

- agent-loop.ts 的 runLoop
- streamAssistantResponse
- executeToolCalls
- Agent.prompt
- Agent.continue

这部分是 Pi Harness 的执行 DNA。

### 5.2 AgentSession 是粘合层

Agent 负责通用执行机制，AgentSession 负责把它接成一个 coding agent：

- 输入预处理
- 工具注册
- System Prompt 重建
- Extension 生命周期
- Session 持久化
- Retry
- Compaction
- Branch Summary
- Steering / Follow-up

### 5.3 SessionManager 不是普通数据库层

SessionManager 维护一个 append-only 的会话树：

- 每条 Entry 有 id 和 parentId
- leaf 指向当前会话位置
- 新消息追加为当前 leaf 的子节点
- 分支通过移动 leaf 实现
- Compaction 作为特殊 Entry 保存
- buildSessionContext() 负责构建真正发送给模型的消息

### 5.4 Prompt 是动态构造的

最终 System Prompt 不只是一个静态文件：

```
基础 Prompt
  + append system prompt
  + 当前工具 snippets
  + 工具 guidelines
  + AGENTS.md 等上下文文件
  + Skills
  + 当前工作目录
  + Extension 动态修改
```

需要区分：

- Prompt 的源文件
- Prompt 的组装函数
- Agent.state.systemPrompt
- 最终发送给 Provider 的请求 Payload

## 6. 两条 Harness 路径

Pi 仓库中存在一条新的 AgentHarness 重构路径：

```
packages/agent/src/harness/agent-harness.ts
  ├── 新 Session Tree
  ├── Lane
  ├── Operation Record
  ├── Durable Recovery
  ├── Compaction / Resume
  └── Hooks / Events
```

但当前 AgentHarness 的 prompt、compact、resume 等多个方法仍然返回 HarnessNotImplemented。

因此本阶段：

- 先研究当前可运行主路径
- 把新 AgentHarness 作为后续的架构演化专题
- 不要把两条路径混进同一张主架构图

## 7. 第一轮暂时排除

暂时不深入：

- TUI 组件和渲染
- CLI 参数细节
- HTML 导出
- 包管理器
- 认证 UI
- SQLite 新后端
- 具体 Provider 的 HTTP 实现

这些属于外壳或支撑层，后续如果发现它们改变 Agent 行为，再单独纳入。

## 8. 代码证据入口

- packages/agent/src/agent-loop.ts：主循环、模型调用边界、Tool Call 循环
- packages/agent/src/agent.ts：Agent 状态、生命周期、队列和事件
- packages/coding-agent/src/core/agent-session.ts：AgentSession、Prompt、Tool、Retry、Compaction
- packages/coding-agent/src/core/system-prompt.ts：System Prompt 组装
- packages/coding-agent/src/core/resource-loader.ts：上下文文件、Skills、Prompt、Extensions 加载
- packages/coding-agent/src/core/session-manager.ts：JSONL Session、Entry Tree、Branch、Compaction
- packages/coding-agent/src/core/model-runtime.ts：Provider、认证和 streamSimple

后续每个组件笔记都要回到这些代码入口确认，不把本文件的架构图当作最终证据。
