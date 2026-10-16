# Pi Harness 组件 05：Tool System

> Tool System 框架状态：已共同阅读并确认。
>
> Concrete Tools 状态：部分完成。具体工具清单和逐项拆解见 `05_tools/`。

Tool System 负责把“模型可以调用什么”连接到“宿主实际执行什么”，并把执行结果重新放回 Agent Context。

## 1. 核心问题

一个 Agent 的工具系统需要同时解决四件事：

~~~text
1. 向模型声明工具
2. 在宿主进程中找到并执行工具
3. 控制参数、权限、并发和错误
4. 把结果变成下一次模型调用可以理解的消息
~~~

Pi 的 Tool System 横跨三个层次：

~~~text
coding-agent
工具定义 / 注册 / 激活 / 扩展 Hook / 内置工具
        ↓
agent-core
参数校验 / 调度 / 执行 / 结果回写
        ↓
pi-ai
把工具定义转换为 OpenAI、Anthropic 等 Provider 格式
~~~

## 2. 总体架构

~~~text
内置工具 / 扩展工具 / SDK Custom Tools
        ↓
ToolDefinition
        ↓
Tool Registry
        ↓
选择 Active Tools
        ├── promptSnippet / promptGuidelines → System Prompt
        └── name / description / parameters → Context.tools
                                                ↓
                                        Provider 工具协议
                                                ↓
                                        模型返回 Tool Call
                                                ↓
                                      prepareArguments()
                                                ↓
                                      参数 Schema 校验
                                                ↓
                                       beforeToolCall()
                                                ↓
                                          execute()
                                                ↓
                                    tool_execution_update
                                                ↓
                                        afterToolCall()
                                                ↓
                                      ToolResultMessage
                                                ↓
                                      写回 Agent Context
                                                ↓
                                       下一次模型调用
~~~

模型只负责提出结构化的 Tool Call；工具查找、参数校验、执行、调度和结果回写都由宿主侧完成。

## 3. ToolDefinition：工具的完整描述

Pi 的 ToolDefinition 不只是一个 execute() 函数，它同时描述模型接口、Prompt 指导、运行行为和 UI 展示：

~~~text
ToolDefinition
├── name                 模型调用时使用的名字
├── label                UI 显示名称
├── description          Provider 工具描述
├── promptSnippet        System Prompt 中的工具简述
├── promptGuidelines     System Prompt 中的使用规则
├── parameters           TypeBox 参数 Schema
├── prepareArguments     校验前的参数兼容处理
├── executionMode        sequential / parallel
├── execute              宿主侧真正执行的函数
├── renderCall           工具调用的 UI 渲染
└── renderResult         工具结果的 UI 渲染
~~~

例如 read 工具：

~~~text
name: read
description: Read the contents of a file...
promptSnippet: Read file contents
promptGuidelines:
  Use read to examine files instead of cat or sed.
parameters:
  path
  offset?
  limit?
execute(): 读取本地文件
~~~

ToolDefinition 的代码定义位于：

~~~text
packages/coding-agent/src/core/extensions/types.ts:451
~~~

## 4. Registry 和 Active Tools

Pi 区分“已经注册的工具”和“当前可以使用的工具”。

~~~text
完整 Tool Registry
├── read
├── bash
├── edit
├── write
├── grep
├── find
├── ls
└── 扩展或 SDK 工具
        ↓ setActiveToolsByName()
当前 Agent.state.tools
~~~

完整 Registry 中的工具不一定全部交给模型。setActiveToolsByName() 会：

1. 按名称查找工具；
2. 忽略不存在的名称；
3. 写入 agent.state.tools；
4. 根据新的工具集合重建 System Prompt；
5. 让变化在下一次 Agent turn 生效。

内置工具在 _buildRuntime() 中创建。扩展通过 registerTool() 注册，SDK 也可以提供 Custom Tools。_refreshToolRegistry() 再把它们合并为最终 Registry。

自定义工具可以覆盖同名内置工具：

~~~text
内置 bash
    ↓
扩展或 SDK 提供同名 bash
    ↓
最终 Registry 使用自定义定义
~~~

相关代码：

~~~text
packages/coding-agent/src/core/agent-session.ts:2588
packages/coding-agent/src/core/agent-session.ts:2681
packages/coding-agent/src/core/agent-session.ts:939
~~~

## 5. 工具进入模型 Context 的双通道

工具定义会通过两条不同的通道进入模型请求：

~~~text
工具名称 / 描述 / 参数 Schema
        ↓
Context.tools
        ↓
Provider 的结构化工具参数
        ↓
模型可以生成 Tool Call
~~~

~~~text
promptSnippet / promptGuidelines
        ↓
System Prompt
        ↓
模型得到工具的使用说明和行为规范
~~~

因此：

~~~text
Context.tools  = 模型能调用什么，以及参数应该长什么样
System Prompt  = 模型什么时候、为什么、如何调用
execute()      = 模型完全看不到的本地实现
~~~

以 read 为例，模型可能同时看到：

~~~text
System Prompt:
  Read file contents
  Use read to examine files instead of cat or sed.

结构化工具参数:
  read(path: string, offset?: number, limit?: number)
~~~

Provider 适配器会把统一的 Context.tools 转换成不同供应商的格式，例如 OpenAI 的 function tool 或 Anthropic 的 input_schema。本地 execute() 函数不会被序列化发送。

## 6. Tool Call 执行链路

模型返回的内容是结构化 Tool Call，而不是一段 JavaScript：

~~~json
{
  "type": "toolCall",
  "name": "read",
  "arguments": {
    "path": "src/agent.ts",
    "offset": 1,
    "limit": 100
  }
}
~~~

Agent Loop 处理这个 Tool Call 的顺序是：

~~~text
1. 根据 name 查找当前 Active Tools
2. 找不到工具 → 生成错误 Tool Result
3. prepareArguments()
4. 按 TypeBox Schema 校验参数
5. beforeToolCall()
6. 调用 tool.execute()
7. 接收 onUpdate() 产生的执行进度事件
8. afterToolCall()
9. 生成 ToolResultMessage
10. 写入当前 Context
~~~

对应代码：

~~~text
查找、准备、校验和 before hook：packages/agent/src/agent-loop.ts:600
实际执行：packages/agent/src/agent-loop.ts:670
after hook：packages/agent/src/agent-loop.ts:713
结果消息：packages/agent/src/agent-loop.ts:777
~~~

如果工具执行抛出异常，Pi 会把异常转换成错误 Tool Result，而不是直接让整个 Agent Loop 崩溃：

~~~text
工具异常
    ↓
isError: true
content: 错误信息
    ↓
模型看到错误
    ↓
模型决定重试、换工具或结束
~~~

如果模型输出因为达到 token 上限而被截断，即使 Tool Call 的参数勉强能解析，Pi 也不会执行这些调用，而是返回错误结果，要求模型重新发出完整调用。

## 7. Tool Result 的内容和用途

工具结果通常包含：

~~~text
AgentToolResult
├── content          返回给模型的文字或图片
├── details          给 UI、日志和渲染器使用的结构化信息
├── usage            工具自身的用量信息
├── addedToolNames   从该消息之后新增的工具名称
└── terminate        是否允许当前 Tool Batch 结束
~~~

content 才是模型下一轮真正需要理解的结果；details 可以保存 UI 展示所需的信息，而不必全部塞进模型 Context。

例如 Bash 工具可以返回：

~~~text
content:
  截断后的命令输出

details:
  truncation
  fullOutputPath
~~~

这样可以同时满足 Context 控制和用户查看完整结果的需要。

工具执行过程中的 onUpdate() 主要产生 tool_execution_update 事件，用于流式 UI 或扩展监听；最终的 content 才会形成 ToolResultMessage 并进入对话历史。

## 8. Hook 和扩展拦截层

Agent Core 提供两个低层 Hook：

~~~text
beforeToolCall
    ↓
允许、阻止，或给出阻止原因

tool.execute()
    ↓
afterToolCall
    ↓
修改 content、details、isError、usage 或 terminate
~~~

在 coding-agent 中，这两个 Hook 被连接到扩展事件：

~~~text
tool_call
tool_result
~~~

因此扩展可以实现：

~~~text
权限检查
用户确认
审计日志
结果重写
图片结果规范化
动态工具注册
~~~

扩展工具还会通过 wrapper 获得 ExtensionContext，所以工具执行本身可以访问扩展运行时提供的上下文和能力。

## 9. 多 Tool Call 的调度

如果模型一次返回多个 Tool Call：

~~~text
assistant message
├── read("a.ts")
├── read("b.ts")
└── grep("TODO")
~~~

Pi 默认采用宿主侧并行调度：

~~~text
准备和校验：顺序进行
实际 execute：允许并行
ToolResultMessage：按模型原始顺序写回
tool_execution_end：可能按实际完成顺序发出
~~~

如果全局 toolExecution 是 sequential，或者某个工具声明了：

~~~ts
executionMode: "sequential"
~~~

则当前 Tool Batch 会按顺序执行。

Tool Batch 只有在所有工具结果都设置了 terminate: true 时才会提前结束；否则 Agent Loop 会把 Tool Results 写入 Context，再发起下一次模型调用。

## 10. Pi 与 Codex 的工具编排差异

Pi 的默认路径是：

~~~text
模型
  ↓
直接返回多个结构化 Tool Call
  ↓
Agent Loop 在宿主侧调度
  ↓
每个 Tool Call 调用一个本地 execute()
~~~

你之前看到的 Codex 路径更特殊，可以抽象为：

~~~text
模型
  ↓
通过 JS / exec 暴露一个额外的工具编排层
  ↓
在编排层中组织多个底层操作
~~~

核心差异是：

~~~text
Pi：工具编排主要由 Agent Loop 固定完成
Codex：工具编排能力被进一步集中到 exec / JS 层
~~~

Pi 的 bash 工具虽然能够执行 Shell 命令，但它只是一个具体工具，不等于一个由模型生成的通用 JavaScript 编排器。

因此，Codex 的编排方式是一个值得单独研究的设计变体，不应直接当成所有 Harness 的标准结构。

## 11. 设计取舍

### 优点

- 工具定义、参数校验和本地执行边界清晰；
- 模型只接触结构化工具协议，不接触宿主代码；
- beforeToolCall 和 afterToolCall 提供权限、安全和结果处理入口；
- 多工具调用可以在宿主侧并行执行；
- 工具错误被转成 Tool Result，模型可以自行恢复；
- 工具输出可以把“给模型的内容”和“给 UI 的详情”分开。

### 代价

- 工具信息同时存在于 Tool Schema 和 System Prompt 两套表达中，需要保持一致；
- Registry、Active Tools、ToolDefinition、AgentTool 分布在不同层；
- 并行执行带来事件顺序和消息顺序的区别；
- 工具结果仍然可能很大，需要依赖工具截断和 Context 管理；
- Pi 没有内置一个通用的模型侧 JavaScript 工具编排语言。

## 12. 组件边界

~~~text
Tool System 负责：
工具注册、激活、声明、校验、拦截、调度、执行和结果回写

Context and Prompt 负责：
工具说明进入 System Prompt，以及工具结果进入 Context 后的裁剪和转换

Agent Loop 负责：
何时调用模型、何时发现 Tool Call、何时继续下一轮

Session and Persistence 负责：
工具调用和 Tool Result 如何持久化、恢复和回放
~~~

### 主要代码入口

~~~text
packages/coding-agent/src/core/tools/index.ts
packages/coding-agent/src/core/extensions/types.ts
packages/coding-agent/src/core/agent-session.ts
packages/coding-agent/src/core/extensions/loader.ts
packages/coding-agent/src/core/extensions/wrapper.ts
packages/agent/src/types.ts
packages/agent/src/agent-loop.ts
packages/ai/src/api/*
~~~
