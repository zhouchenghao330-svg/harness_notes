# Pi Harness：Extensions、Skills 与 Resources

状态：已共同阅读，Code-verified

核心源码：

- packages/coding-agent/src/core/resource-loader.ts
- packages/coding-agent/src/core/extensions/loader.ts
- packages/coding-agent/src/core/extensions/runner.ts
- packages/coding-agent/src/core/extensions/types.ts
- packages/coding-agent/src/core/skills.ts
- packages/coding-agent/src/core/system-prompt.ts
- packages/coding-agent/src/core/agent-session.ts

## 1. 组件结论

Pi 的 Extension 不是普通的“插件列表”，而是一套可以介入 Harness 运行时的扩展协议。

Pi Core 在关键生命周期位置预留 Hook，Extension 通过 Extension API 注册 Handler，ExtensionRunner 负责按顺序调用 Handler，AgentSession 再根据 Handler 的返回值决定是否继续执行默认逻辑、取消操作，或者使用 Extension 提供的结果。

因此，Extension 可以影响：

- 用户输入；
- System Prompt；
- Context；
- Tool Call 和 Tool Result；
- Compaction 和 Branch Summary；
- Provider Request；
- Model 和 Provider；
- Session 生命周期；
- UI、命令、快捷键和渲染；
- Skills、Prompt Templates 和 Themes 的动态发现。

但 Extension 不是任意修改私有源码。Session 持久化、状态重建和生命周期主流程仍然由 Pi Core 掌握。

## 2. 总体架构

### 2.1 启动阶段

~~~text
用户配置、项目目录、Package、CLI 参数
                │
                ▼
        DefaultResourceLoader
                │
      ┌─────────┼─────────┬─────────┬─────────┐
      ▼         ▼         ▼         ▼         ▼
  Extensions  Skills  Prompt     Themes   Context Files
                       Templates
      │         │         │         │         │
      └─────────┴─────────┴─────────┴─────────┘
                │
                ▼
          AgentSession
                │
                ▼
        ExtensionRunner
                │
                ▼
          Agent Runtime
~~~

### 2.2 运行阶段

~~~text
用户输入
  ↓
input Hook
  ↓
Extension Command 检查
  ↓
Skill / Prompt Template 展开
  ↓
before_agent_start
  ↓
Agent Loop
  ├── context
  ├── before_provider_headers
  ├── before_provider_request
  ├── Tool Call
  ├── Tool Result
  └── Session / Compaction / Tree Hooks
  ↓
Agent 继续、停止或等待下一条消息
~~~

## 3. ResourceLoader：资源发现和装配层

ResourceLoader 负责把分散在不同位置的资源整理成当前 Session 可用的集合。

它管理的资源包括：

- Extensions；
- Skills；
- Prompt Templates；
- Themes；
- AGENTS.md、CLAUDE.md 等 Context Files；
- System Prompt；
- Append System Prompt。

它不是 Agent Loop，也不是 Context Manager。它主要负责：

1. 找到资源；
2. 判断资源是否启用；
3. 解析资源；
4. 处理来源和优先级；
5. 去重和冲突诊断；
6. 把资源交给 AgentSession；
7. 支持 reload 和 Extension 动态追加资源。

### 3.1 资源来源

资源可以来自：

~~~text
全局目录
  ~/.pi/agent/...

项目目录
  .pi/...

项目祖先目录
  .agents/skills/...

Pi Package
  package.json 的 pi manifest

settings.json
  显式配置路径

CLI
  --extension
  --skill
  以及其他临时路径
~~~

DefaultResourceLoader.reload() 会先解析 Package 和配置，再决定哪些资源启用。

### 3.2 资源来源信息

Pi 不只保存资源本身，还保存 Resource Metadata，例如：

- 来源是 local、package、cli 还是 extension；
- 来源属于 user、project 还是 temporary；
- 资源来自哪个 Package；
- 资源的 baseDir；
- 发生冲突时哪个资源获胜。

这使得 TUI、RPC、命令列表和诊断信息可以知道：

~~~text
这个 Skill / Tool / Prompt 是从哪里来的？
~~~

### 3.3 动态追加资源

Extension 可以监听 resources_discover：

~~~text
session_start
    ↓
resources_discover
    ↓
Extension 返回 skillPaths / promptPaths / themePaths
    ↓
AgentSession.buildExtensionResourcePaths()
    ↓
ResourceLoader.extendResources()
    ↓
重新装配 Skills、Prompts、Themes
    ↓
重建 System Prompt 和命令列表
~~~

这意味着 Extension 不只能注册代码，也能动态提供一组 Markdown 或 Theme 资源。

## 4. Extension 的加载模型

### 4.1 Extension Factory

Extension 是一个 TypeScript 或 JavaScript 模块，默认导出一个 Factory：

~~~text
Extension Module
    ↓
default function(pi: ExtensionAPI)
    ↓
注册 Handler、Tool、Command、Flag、Renderer
~~~

Factory 可以是异步的。Pi 会等待异步 Factory 完成后再继续 Session 启动。

这允许 Extension 在加载阶段完成一次性初始化，例如：

- 拉取远程模型列表；
- 注册 Provider；
- 读取配置；
- 计算静态资源路径。

### 4.2 Extension 内部结构

Loader 会为每个 Extension 创建一个结构化对象：

~~~text
Extension
  ├── handlers
  ├── tools
  ├── commands
  ├── flags
  ├── shortcuts
  ├── messageRenderers
  ├── entryRenderers
  └── markdownTransformer
~~~

pi.on() 注册的不是一个全局回调，而是写入当前 Extension 的 handlers Map。

registerTool()、registerCommand() 等方法也只是先写入当前 Extension 的注册表。

## 5. Extension Runtime：为什么 Handler 可以调用 Session

Extension 在加载时拿到的 pi API，并不是直接持有 AgentSession。

Loader 会先创建一个带有临时 Stub 的 ExtensionRuntime：

~~~text
Extension Loading 阶段
  ├── registerTool() 可以注册
  ├── registerCommand() 可以注册
  ├── registerProvider() 先进入队列
  └── sendMessage() 等 Session Action 暂时不可用
~~~

等 AgentSession 创建完成后，AgentSession 通过 bindCore() 把真正的 Action 注入 Runtime：

~~~text
ExtensionRuntime
  ├── sendMessage       → AgentSession.sendCustomMessage
  ├── sendUserMessage   → AgentSession.sendUserMessage
  ├── appendEntry       → SessionManager.appendCustomEntry
  ├── setActiveTools    → AgentSession Tool Registry
  ├── setModel          → ModelRuntime
  ├── compact           → AgentSession.compact
  └── getSystemPrompt   → 当前有效 System Prompt
~~~

这是一种延迟绑定：

~~~text
先加载 Extension
    ↓
收集注册信息
    ↓
创建 AgentSession
    ↓
绑定真实运行时 Action
    ↓
Extension Handler 获得完整能力
~~~

这样 Extension 代码不需要依赖具体 TUI、RPC 或 CLI 实现。

## 6. ExtensionRunner：事件分发层

ExtensionRunner 是 Extension 和 AgentSession 之间的中间层。

它主要负责：

- 按 Extension 加载顺序调用 Handler；
- 创建当前时刻的 ExtensionContext；
- 对特定事件解释 Handler 返回值；
- 处理多个 Extension 的串联；
- 捕获 Extension 错误并报告；
- 在 reload 或 Session 替换后让旧 Context 失效。

基本模型：

~~~text
AgentSession
    │ emit(event)
    ▼
ExtensionRunner
    ├── Extension A Handler
    ├── Extension B Handler
    └── Extension C Handler
    │
    ▼
返回处理结果
~~~

Extension 本身不直接调用所有其他 Extension。所有 Handler 的组合逻辑集中在 Runner 内。

### 6.1 Handler 的执行顺序

默认按照 Extension 的加载顺序执行：

~~~text
Extension A
    ↓
Extension B
    ↓
Extension C
~~~

对于普通事件，后面的 Handler 通常可以看到前面 Handler 修改后的数据，或者至少看到同一个生命周期事件。

对于有特殊返回协议的事件，Runner 会根据事件类型决定：

- 是否短路；
- 是否把结果传给下一个 Handler；
- 是否保存最后一个结果；
- 是否忽略返回值。

### 6.2 错误处理

大部分事件 Handler 抛错时，Runner 会：

1. 捕获异常；
2. 记录 extensionPath 和 event；
3. 通知 Extension Error Listener；
4. 继续执行后续 Handler 或默认流程。

这避免一个可选 Extension 直接破坏整个 Agent Runtime。

但如果某个 Handler 返回了明确的阻止结果，例如 Tool Call 的 block，Runner 会按该事件的协议停止对应流程。

## 7. Hook 不是修改内部函数，而是返回控制协议

理解 Pi Extension 最重要的一点是：

~~~text
Hook = 事件 + Handler + 返回协议
~~~

Extension 不需要修改 Pi 内部的 compact 函数。Pi Core 已经在关键位置写好了：

~~~text
准备数据
  ↓
调用 Hook
  ↓
解释 Hook 返回值
  ↓
继续默认逻辑，或者使用替代结果
~~~

因此，不同 Hook 的能力取决于 Pi Core 为这个 Hook 定义了什么返回类型。

## 8. Compaction Hook：完整调用链

### 8.1 手动 Compact

用户执行 /compact 时：

~~~text
AgentSession.compact()
  ↓
abort 当前 Agent 操作
  ↓
读取当前 Branch
  ↓
prepareCompaction()
  ↓
触发 session_before_compact
  ↓
解释 Extension 返回值
  ↓
调用 Extension 结果或默认 compact
  ↓
SessionManager.appendCompaction()
  ↓
重建 Agent.state.messages
  ↓
触发 session_compact
  ↓
返回 CompactionResult
~~~

### 8.2 自动 Compact

自动 Compact 由两种情况触发：

- Context 接近阈值；
- Provider 返回 Context Overflow 或可恢复的截断错误。

自动路径同样会触发 session_before_compact，但会额外携带：

- reason：manual、threshold 或 overflow；
- willRetry：Overflow 后是否需要继续原来的 Agent Turn。

~~~text
Agent End
  ↓
检测 Context 使用量或 Overflow
  ↓
_runAutoCompaction()
  ↓
session_before_compact
  ↓
Compact
  ↓
如果 willRetry = true
  → 删除失败响应
  → Agent 继续原来的 Turn
~~~

### 8.3 CompactionPreparation

Extension 不需要自己重新计算哪些消息应该被压缩。Pi 先调用 prepareCompaction()，然后把准备结果交给 Extension：

~~~text
CompactionPreparation
  ├── firstKeptEntryId
  ├── messagesToSummarize
  ├── turnPrefixMessages
  ├── isSplitTurn
  ├── tokensBefore
  ├── previousSummary
  ├── fileOps
  └── settings
~~~

其中：

- messagesToSummarize：会被 Summary 替代的历史消息；
- turnPrefixMessages：当前过大的 Turn 被拆分时，需要额外总结的前半段；
- firstKeptEntryId：压缩后保留消息的起点；
- previousSummary：之前的 Compaction Summary；
- fileOps：从历史 Tool Call 中提取出的读取文件和修改文件；
- tokensBefore：压缩前的 Context Token 估算；
- settings：当前 Compaction 配置。

这个设计把“确定压缩范围”和“生成 Summary”分开了。

## 9. session_before_compact 的三种结果

### 9.1 返回 cancel：取消

~~~typescript
pi.on("session_before_compact", async () => {
  return { cancel: true };
});
~~~

手动 Compact 会以 Compaction cancelled 结束。

自动 Compact 会发出 aborted 的 compaction 结果，并停止这次自动压缩。

### 9.2 返回 compaction：替换 Summary 生成逻辑

~~~typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, signal } = event;
  const model = ctx.modelRegistry.find("google", "gemini-2.5-flash");

  if (!model) {
    return;
  }

  const summary = await createCustomSummary(
    preparation.messagesToSummarize,
    preparation.previousSummary,
    model,
    signal,
  );

  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      details: {
        source: "custom-extension",
      },
    },
  };
});
~~~

返回 compaction 后，AgentSession 不再调用默认的 _runDefaultCompaction()。

但是这并不代表 Extension 接管了整个 Session Compact 流程。Pi Core 仍然负责：

- 保存 Compaction Entry；
- 重新构建 Session Context；
- 更新 Agent.state.messages；
- 触发 session_compact；
- 处理 Overflow 后的 retry；
- 生成最终的 CompactionResult。

因此，Extension 替换的是：

~~~text
Summary 内容的生成
~~~

而不是：

~~~text
整个 Session 状态迁移和持久化流程
~~~

### 9.3 返回 undefined：不干预，执行默认 Compact

~~~typescript
pi.on("session_before_compact", async (event) => {
  await saveCheckpoint(event.preparation.fileOps);

  // 不返回 compaction
  // 默认 Compact 继续执行
});
~~~

这里的 undefined 不是配置缺失，而是一个明确的控制语义：

~~~text
当前 Extension 没有替换结果
    ↓
AgentSession 继续执行默认逻辑
~~~

## 10. Compact 前置、替换和后置逻辑

### 10.1 前置逻辑

适合：

- 保存 Git Checkpoint；
- 写入外部索引；
- 记录文件操作；
- 统计 Context；
- 给用户发送通知；
- 给自定义 Summary 准备数据。

实现方式：

~~~text
session_before_compact
  → 执行副作用
  → 返回 undefined
  → 默认 Compact
~~~

### 10.2 替换逻辑

适合：

- 使用另一个模型生成 Summary；
- 使用结构化 Summary；
- 保留 Artifact Index；
- 改变 Summary 的关注重点；
- 把全部历史压成一个 Summary；
- 实现领域专用记忆格式。

实现方式：

~~~text
session_before_compact
  → 读取 CompactionPreparation
  → 自己生成 CompactionResult
  → 返回 compaction
  → Pi Core 保存并重建状态
~~~

### 10.3 后置逻辑

适合：

- 读取已经生成的 Summary；
- 把 Summary 同步到外部系统；
- 更新 UI；
- 记录压缩成功；
- 统计 Extension Summary 的 Token 用量。

实现方式：

~~~text
session_compact
  → 已经保存 Compaction Entry
  → 读取 compactionEntry
  → 执行后置逻辑
~~~

### 10.4 失败逻辑

~~~typescript
pi.on("session_compact_failed", async (event) => {
  // event.reason
  // event.errorMessage
  // event.aborted
  // event.willRetry
  // event.fromExtension
});
~~~

这个事件适合做监控、清理和错误记录。

## 11. Pi 如何支持不同 Hook 的不同效果

不同 Hook 并不是共享同一种返回值。每个 Hook 的语义由类型和 AgentSession 调用代码共同定义。

| Hook | 返回或修改方式 | 效果 |
| --- | --- | --- |
| session_before_compact | cancel / compaction / undefined | 取消、替换或继续默认 Compact |
| session_before_tree | cancel / summary / undefined | 取消 Branch 或提供 Branch Summary |
| tool_call | 修改 input、返回 block | 修改参数或阻止工具 |
| tool_result | 返回 content、details、isError、usage | 修改工具结果 |
| context | 返回 messages | 修改本次 LLM Context |
| input | continue / transform / handled | 放行、改写或拦截输入 |
| before_agent_start | message / systemPrompt | 注入消息或修改本轮 System Prompt |
| resources_discover | skillPaths / promptPaths / themePaths | 动态增加资源 |
| before_provider_request | 返回新 payload | 修改 Provider Payload |
| before_provider_headers | 原地修改 headers | 修改请求 Header |
| session_compact | 返回值被忽略 | 观察已完成的 Compact |

可以把这些 Hook 分成几类：

### 11.1 Veto / Replacement Hook

这类 Hook 允许取消或者替换核心行为：

- session_before_compact；
- session_before_tree；
- tool_call。

### 11.2 Transform Hook

这类 Hook 接收一个值，再返回修改后的值：

- context；
- input；
- tool_result；
- before_provider_request。

### 11.3 In-place Mutation Hook

这类 Hook 直接修改事件对象：

- before_provider_headers；
- tool_call 的 input。

### 11.4 Notification Hook

这类 Hook 主要用于观察状态，返回值通常被忽略：

- session_compact；
- session_compact_failed；
- agent_start；
- agent_end；
- turn_start；
- turn_end。

## 12. 多个 Extension 的组合规则

ExtensionRunner 通常按照加载顺序运行：

~~~text
Extension A
  ↓
Extension B
  ↓
Extension C
~~~

对于 session_before_compact：

- Handler 返回 undefined：继续执行；
- Handler 返回 cancel：立即取消；
- Handler 返回 compaction：提供替代结果；
- 后面 Handler 返回新的 compaction 时，可能覆盖前面的结果。

因此，多个 Extension 同时拥有同一个“替换权”时，应该由 Extension 作者约定优先级。

比较稳定的组合方式是：

~~~text
Extension A：只做前置检查
Extension B：唯一的 Summary 替换者
Extension C：只做后置记录
~~~

这比多个 Extension 同时重写 Summary 更容易预测。

## 13. AgentSession 如何接入 Extension

AgentSession 创建 ExtensionRunner 后，会绑定核心 Action 和 Agent Hook。

### 13.1 Tool Hook

~~~text
Agent.beforeToolCall
  → ExtensionRunner.emitToolCall

Agent.afterToolCall
  → ExtensionRunner.emitToolResult
~~~

### 13.2 Context 和 Provider Hook

~~~text
Agent transformContext
  → ExtensionRunner.emitContext

Model Runtime onPayload
  → ExtensionRunner.emitBeforeProviderRequest

Model Runtime transformHeaders
  → ExtensionRunner.emitBeforeProviderHeaders

Model Runtime onResponse
  → ExtensionRunner.emit(after_provider_response)
~~~

### 13.3 Session Hook

~~~text
AgentSession
  ├── session_start
  ├── resources_discover
  ├── session_before_compact
  ├── session_compact
  ├── session_compact_failed
  ├── session_before_tree
  ├── session_tree
  └── session_shutdown
~~~

这说明 Extension 并不是只挂在 TUI 外层，而是被接入了 AgentSession、Agent Loop、Tool System 和 Model Runtime 的多个边界。

## 14. Skills：渐进式加载

Skill 是 Markdown 能力包，不是可执行 Extension。

Pi 启动时通常只读取 Skill 的元数据：

~~~text
Skill
  ├── name
  ├── description
  ├── filePath
  ├── baseDir
  └── disableModelInvocation
~~~

System Prompt 中只放名称、描述和位置：

~~~text
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
~~~

模型判断任务匹配后，再使用 read 工具读取完整 SKILL.md。

因此 Skill 的加载链是：

~~~text
启动扫描
  ↓
读取元数据
  ↓
元数据进入 System Prompt
  ↓
模型判断是否需要
  ↓
read 工具读取完整内容
  ↓
按照 Skill 执行复杂任务
~~~

这就是 Progressive Disclosure：

~~~text
始终保留短描述
按需加载长指令、脚本和参考资料
~~~

用户也可以通过 /skill:name 显式触发。显式触发时，Pi 读取完整文件并把它展开为当前用户消息的一部分。

disable-model-invocation 为 true 时，Skill 不会出现在模型可见的 available_skills 中，但仍然可以通过用户命令显式调用。

## 15. Prompt Templates：用户主动调用的 Prompt 宏

Prompt Template 是 Markdown 形式的自定义 Slash Command。

例如 prompts/review.md：

~~~markdown
---
description: Review a piece of code
argument-hint: <question>
---

请按照代码审查流程分析下面的问题：

$ARGUMENTS

要求：
1. 找到相关实现
2. 判断是否存在问题
3. 给出具体代码证据
~~~

用户输入：

~~~text
/review 看下 block 生成限制数是不是有问题
~~~

Pi 会把它展开成普通用户消息：

~~~text
读取 prompts/review.md
  ↓
解析参数
  ↓
替换 $ARGUMENTS
  ↓
生成完整用户消息
  ↓
进入 Agent Loop
~~~

常用参数：

~~~text
$1            第一个参数
$2            第二个参数
$@            所有参数
$ARGUMENTS    所有参数
~~~

Prompt Template 和 Skill 的区别：

| | Prompt Template | Skill |
| --- | --- | --- |
| 触发 | 用户主动输入 /name | 模型判断或用户输入 /skill:name |
| 加载 | 立即展开完整 Prompt | 默认先只暴露元数据 |
| 作用 | 固定工作流和 Prompt 快捷命令 | 复杂能力包和按需指令 |
| 内容位置 | 进入当前用户消息 | 通常由 read 工具按需读取 |
| 执行代码 | 否 | 否 |

Prompt Template 也不同于 Extension Command：

~~~text
Extension Command
  /deploy
  → 执行 TypeScript
  → 可以访问 UI、Session、Model 和工具

Prompt Template
  /review
  → 展开 Markdown
  → 作为普通用户消息进入 Agent
~~~

## 16. Trust、Reload 和生命周期

### 16.1 Project Trust

项目本地 Extension 可能执行任意代码，因此 Pi 在加载项目资源前有 Trust 阶段。

在 Trust 尚未确定时，Pi 只加载：

- 全局 Extension；
- CLI 显式提供的 Extension；
- 不受项目资源信任控制的 Context。

这些预先加载的 Extension 可以监听 project_trust，参与决定是否信任项目。

Trust 不是沙箱。Extension 仍然拥有当前用户进程的系统权限。

### 16.2 Reload

执行 /reload 时：

~~~text
旧 Extension
  ↓ session_shutdown
旧 Runtime invalidate
  ↓
重新加载资源和 Extension
  ↓
创建新的 ExtensionRunner
  ↓
session_start(reason = reload)
  ↓
resources_discover(reason = reload)
~~~

旧 Extension Context 会被标记为 stale。这样可以避免 Extension 在 reload、fork 或切换 Session 后继续使用过期的 Runner、Session 或闭包状态。

### 16.3 Session 替换

newSession、fork、switchSession 也会销毁旧 Extension Runtime，并为新 Session 建立新的 Extension 实例。

因此长生命周期资源应该：

- 在 session_start 中创建；
- 在 session_shutdown 中清理；
- 不要依赖旧 Context 跨 Session 使用。

## 17. 设计边界

Pi 的 Extension 设计可以总结为：

~~~text
Extension 控制“可变策略”
  ├── 是否允许
  ├── 如何转换
  ├── 是否替换
  ├── 如何呈现
  └── 如何补充能力

Pi Core 控制“稳定骨架”
  ├── Agent Loop 主流程
  ├── Session 持久化
  ├── Agent State 重建
  ├── Compaction Entry 写入
  ├── 生命周期顺序
  └── 核心数据结构
~~~

所以它不是 Monkey Patch，而是受控的可插拔控制流。

这种设计的关键价值是：

- 核心保持小；
- 工作流可以由用户自己定义；
- 不同 Extension 可以组合；
- 默认行为仍然存在；
- Extension 失败不会必然破坏整个 Agent；
- 核心可以保证 Session 和状态一致性。

代价是：

- Hook 数量和返回协议会增加学习成本；
- 多个 Extension 之间可能发生优先级冲突；
- Extension 需要理解 Agent 生命周期；
- 动态资源、Trust、Reload 和 stale Context 会带来额外复杂度；
- 任意 Extension 代码都可能改变 Agent 行为，安全责任较高。

## 18. 本章问题的直接答案

### 问题一：是不是作者预先内置了很多 Hook？

是。Pi Core 在输入、Context、Tool、Provider、Session、Compaction、Branch、UI 等关键边界预留了 Hook。

### 问题二：Extension 是不是修改 Hook 内部逻辑？

不是。Extension 注册 Handler，Core 调用 Handler，并解释 Handler 的返回值。

### 问题三：为什么不同 Hook 有不同效果？

因为每个 Hook 都定义了自己的事件类型和返回协议。Core 根据协议决定是继续、转换、取消、替换还是只通知。

### 问题四：session_before_compact 的三种返回是什么意思？

~~~text
{ cancel: true }
  → 取消 Compact

{ compaction: {...} }
  → 使用 Extension 提供的 Summary 结果

undefined
  → Extension 不干预，继续 Pi 默认 Compact
~~~

### 问题五：Extension 替换了整个 Compact 吗？

没有。它替换的是 Summary 内容生成。Compaction Entry 保存、Session 重建、Agent State 更新和后续 Retry 仍由 Pi Core 负责。

## 19. 关键源码定位

- ResourceLoader 接口和资源读取：packages/coding-agent/src/core/resource-loader.ts:34-50
- ResourceLoader.reload、Trust 和资源装配：packages/coding-agent/src/core/resource-loader.ts:388-545
- Extension 动态追加 Skills、Prompts、Themes：packages/coding-agent/src/core/resource-loader.ts:340-376
- Skill 扫描和元数据加载：packages/coding-agent/src/core/skills.ts:44-352、407-507
- Skill System Prompt 格式化：packages/coding-agent/src/core/skills.ts:355-381
- Prompt Template 加载和参数替换：packages/coding-agent/src/core/prompt-templates.ts:1-285
- Extension Runtime Stub 和延迟绑定：packages/coding-agent/src/core/extensions/loader.ts:186-245
- Extension API 注册和 Runtime Action：packages/coding-agent/src/core/extensions/loader.ts:254-459
- Extension 模块加载和发现：packages/coding-agent/src/core/extensions/loader.ts:524-806
- Extension 类型、Context 和 Tool 定义：packages/coding-agent/src/core/extensions/types.ts:309-449
- Session、Agent、Tool、Provider 事件定义：packages/coding-agent/src/core/extensions/types.ts:541-629、724-824
- ExtensionAPI 和注册接口：packages/coding-agent/src/core/extensions/types.ts:1230-1360
- ExtensionRunner 事件分发：packages/coding-agent/src/core/extensions/runner.ts:801-1235
- AgentSession 创建 Runner、绑定 Core 和动态资源：packages/coding-agent/src/core/agent-session.ts:2362-2410、2705-2760
- AgentSession Tool、Context、Provider Hook 接入：packages/coding-agent/src/core/agent-session.ts:479-540、2590-2680
- Manual Compact Hook 和默认分支：packages/coding-agent/src/core/agent-session.ts:1822-2010
- Automatic Compact Hook 和 Overflow Retry：packages/coding-agent/src/core/agent-session.ts:2157-2345
- CompactionPreparation 和底层 Summary 生成：packages/coding-agent/src/core/compaction/compaction.ts:733-930
- 官方自定义 Compact 示例：packages/coding-agent/examples/extensions/custom-compaction.ts:1-104
