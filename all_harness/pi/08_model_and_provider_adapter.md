# Pi Harness 组件 08：Model Runtime and Provider Adapter

> 状态：已共同阅读并确认。

## 1. 核心结论

这一部分主要是 Harness 与外部模型服务之间的调用基础设施，不是 Agent 的决策逻辑。

它负责：

- 模型选择和能力描述；
- Provider 认证；
- API 请求协议转换；
- Tool Schema 和消息格式转换；
- SSE / WebSocket 流式响应解析；
- Provider 事件到统一 Agent 事件的归一化。

整体调用链：

~~~text
Agent Loop
  决定什么时候调用模型、是否执行工具、是否继续
        ↓
Model Runtime
  选择模型、解析认证、准备请求
        ↓
Provider
  按 provider 管理模型和认证
        ↓
API Adapter
  按 api 处理具体协议
        ↓
OpenAI / Anthropic / Google / 其他模型服务
        ↓
统一的 AssistantMessageEventStream
~~~

Agent Core 只需要知道：

~~~text
Model
Context
StreamFn
AssistantMessageEventStream
~~~

它不需要知道 HTTP、SSE、Anthropic Messages、OpenAI Responses 或具体的请求字段。

## 2. 组件边界

主要代码：

~~~text
packages/ai/src/types.ts
  定义 Model、Context、Provider、请求选项和事件协议

packages/coding-agent/src/core/model-runtime.ts
  管理模型目录、Provider、认证、请求准备和派发

packages/coding-agent/src/core/model-resolver.ts
  将模型模式解析成实际 Model

packages/ai/src/models.ts
  实现 Models 集合、Provider 调度和认证应用

packages/ai/src/api/
  实现 OpenAI、Anthropic、Google 等具体 API Adapter

packages/ai/src/api/lazy.ts
  延迟加载 Adapter，并把初始化错误转为统一 stream error
~~~

可以把它看成五个层次：

~~~text
Model Descriptor
        ↓
ModelRuntime
        ↓
Models / Provider
        ↓
API Adapter
        ↓
Provider Event Normalizer
~~~

## 3. Model Descriptor：模型的能力合同

Pi 的 Model 不是一个客户端对象，而是一份模型能力描述：

~~~text
Model
├── id
├── name
├── provider
├── api
├── baseUrl
├── reasoning
├── thinkingLevelMap
├── input
├── cost
├── contextWindow
├── maxTokens
├── samplingParams
├── headers
└── compat
~~~

其中最重要的字段：

| 字段 | 作用 |
| --- | --- |
| id | Provider 内部的模型 ID |
| provider | 认证、模型目录和服务归属 |
| api | 实际采用的协议 Adapter |
| baseUrl | 请求目标地址 |
| reasoning | 是否支持 reasoning / thinking |
| input | 支持 text、image 或两者 |
| contextWindow | Context 上限 |
| maxTokens | 输出上限 |
| cost | 输入、输出和缓存计费 |
| compat | 同一协议下的兼容差异 |

Model 是后续请求的输入合同。ModelRuntime、Provider 和 API Adapter 都依据它工作。

## 4. provider 和 api 的双层设计

这是这一部分最重要的设计。

~~~text
provider
  = 谁提供服务、使用什么账号和认证

api
  = 使用哪种请求协议和消息格式
~~~

例如：

~~~text
provider = github-copilot
api      = openai-responses
model    = gpt-5.4
~~~

同一个 Provider 可以提供多个 API：

~~~text
github-copilot
├── anthropic-messages
├── openai-completions
└── openai-responses
~~~

因此，不能只根据 Provider 判断请求格式。最终请求应该由 model.api 选择 Adapter。

反过来，多个 Provider 也可以复用同一个 API Adapter：

~~~text
openai-completions Adapter
├── openai
├── deepseek
├── openrouter
├── groq
└── 自定义 OpenAI-compatible Provider
~~~

这个设计把：

- 服务归属和认证；
- 请求协议和消息转换；

拆成了两个独立维度。

## 5. ModelRuntime：模型调用运行时

ModelRuntime 负责把静态模型描述变成可以实际调用的运行时模型。

创建过程：

~~~text
ModelRuntime.create()
    ↓
加载 auth.json
    ↓
加载 models.json
    ↓
加载内置 Provider
    ↓
加载或恢复动态模型目录
    ↓
组合 models.json / Extension Provider 覆盖
    ↓
检查 Provider 可用性
    ↓
生成 all / available 模型快照
~~~

它维护两类模型视图：

~~~text
all
  所有已知模型

available
  Provider 已完成认证、当前可使用的模型
~~~

模型选择时，ModelResolver 通常基于 available 模型：

~~~text
模型模式
  例如 provider/model、model、glob、model:thinking-level
        ↓
ModelResolver
        ↓
实际 Model 对象 + thinking level
~~~

Session 恢复时，Pi 还会根据 Session 中保存的 provider/modelId 重新查找当前 Model。

因此：

~~~text
模型选择
  只产生 Model Descriptor

模型调用
  再由 ModelRuntime 使用这个 Descriptor 完成
~~~

## 6. 一次模型调用的完整路径

在 createAgentSession 中，Agent 的 streamFn 连接到 ModelRuntime：

~~~text
createAgentSession()
    ↓
创建 ModelRuntime
    ↓
选择 model
    ↓
创建 Agent
    ↓
Agent.streamFunction = modelRuntime.streamSimple
~~~

一次 Agent Loop 请求：

~~~text
Agent Loop
    ↓
transformContext()
    ↓
convertToLlm()
    ↓
构造 Context {
  systemPrompt,
  messages,
  tools
}
    ↓
streamFunction(model, context, options)
    ↓
ModelRuntime.prepareRequest()
    ↓
解析 Provider 和认证
    ↓
调用 provider.streamSimple()
    ↓
API Adapter
    ↓
返回统一事件流
~~~

Agent Loop 只接收统一事件，不直接处理 Provider 原始响应。

## 7. 认证和请求准备

ModelRuntime 在真正发送请求前，会完成一层统一的请求准备：

~~~text
prepareRequest(model, options)
    ↓
根据 model.provider 找 Provider
    ↓
解析 API Key / OAuth / 环境认证
    ↓
合并认证 headers 和请求 headers
    ↓
合并 Provider 环境变量和请求 env
    ↓
应用 transformHeaders
    ↓
使用认证返回的 baseUrl 覆盖 Model baseUrl
    ↓
得到可发送的 requestModel + requestOptions
~~~

认证来源可能包括：

- 运行时注入的 API Key；
- Session 或配置文件中的 Credential；
- 环境变量；
- OAuth Credential；
- Provider 自己的环境认证，例如 AWS Profile 或 ADC；
- models.json 或 Extension Provider 配置。

Provider 未配置时，ModelRuntime 不会让 Adapter 自己猜测认证，而是返回统一的 auth error。

请求级选项可以覆盖部分默认值：

~~~text
Provider 默认值
    ↓
认证解析结果
    ↓
Model headers / baseUrl
    ↓
本次请求 options
    ↓
transformHeaders
    ↓
最终请求
~~~

这样可以让统一的认证层和 Provider-specific 请求逻辑保持分离。

## 8. API Adapter 具体负责什么

API Adapter 接收统一输入：

~~~text
model
context.systemPrompt
context.messages
context.tools
options
~~~

然后转换成 Provider API 的真实请求。

它通常负责：

- systemPrompt 到 Provider system/developer 字段的转换；
- user、assistant、toolResult 消息转换；
- Tool Schema 转换；
- Tool Call ID 规范化；
- image content 转换；
- thinking / reasoning 参数映射；
- maxTokens 和 sampling 参数映射；
- prompt cache 参数映射；
- Provider-specific headers；
- request payload hook；
- HTTP response hook。

例如 OpenAI Completions Adapter 的处理路径：

~~~text
Context
    ↓
transformMessages()
    ↓
system / developer / user / assistant / tool
    ↓
Chat Completion payload
    ↓
client.chat.completions.create()
~~~

Anthropic Adapter 的目标协议不同，但外部接口仍然是：

~~~text
Model + Context + Options
    ↓
Anthropic Messages payload
    ↓
AssistantMessageEventStream
~~~

## 9. Provider-specific compat

即使两个 Model 使用同一个 api，也不一定完全兼容。

例如 OpenAI-compatible 服务之间可能存在差异：

~~~text
服务 A
  使用 max_completion_tokens

服务 B
  使用 max_tokens

服务 C
  支持 developer role

服务 D
  只支持 system role
~~~

Pi 通过 Model.compat 表达这些协议差异：

- 是否支持 developer role；
- 是否支持 store；
- 是否支持流式 usage；
- 使用哪个 max tokens 字段；
- Tool Result 是否需要 name；
- 是否支持 reasoning_effort；
- 是否支持 grammar tool；
- prompt cache 的格式；
- 是否需要特殊的 finish reason 推断。

所以 API Adapter 不是简单的 URL 拼接器，而是：

~~~text
统一语义
    ↓
API 协议转换
    ↓
具体服务兼容处理
~~~

## 10. 流式处理和事件归一化

不同 Provider 的原始流格式不同：

~~~text
OpenAI Completions
  chat.completion.chunk

OpenAI Responses
  response.output_text.delta

Anthropic
  content_block_delta

Google
  candidates / parts / finishReason
~~~

Pi 将它们统一为 AssistantMessageEventStream：

~~~text
start
text_start
text_delta
text_end
thinking_start
thinking_delta
thinking_end
toolcall_start
toolcall_delta
toolcall_end
done
error
~~~

Agent Loop 因此可以统一处理：

~~~text
text_delta
    → 更新 streamingMessage

thinking_delta
    → 更新 thinking 内容

toolcall_delta
    → 累积 Tool Call 参数

done
    → 得到完整 AssistantMessage

error
    → 得到 stopReason = error 或 aborted
~~~

Adapter 还负责把 Provider 的 finish reason 转换为 Pi 的 stopReason：

~~~text
stop
length
toolUse
deferred
error
aborted
~~~

这一步是 Agent Loop 能够保持 Provider-agnostic 的关键。

## 11. streamSimple 和 stream

Pi 暴露两个层次的调用接口。

### streamSimple

面向 Agent 的 Provider-neutral 接口：

~~~text
SimpleStreamOptions
├── reasoning
├── toolChoice
├── maxTokens
├── sessionId
├── transport
└── thinkingBudgets
~~~

Adapter 再把它转换成具体协议：

~~~text
reasoning = high
    ↓
OpenAI: reasoning_effort

reasoning = high
    ↓
Anthropic: effort 或 thinkingBudgetTokens

reasoning = high
    ↓
Google: Provider-specific thinking 参数
~~~

Agent 使用 streamSimple，是因为 Agent 不应该依赖每个 Provider 的专用参数。

### stream

面向底层调用方的 API-specific 接口：

~~~text
stream(model, context, providerSpecificOptions)
~~~

它允许调用方直接使用某个 API 的专用能力。

因此：

~~~text
Agent Core
  → streamSimple()

底层 API / 高级 Provider 集成
  → stream()
~~~

## 12. Lazy Stream 和错误统一

认证解析和 API Adapter 加载可能是异步的，但调用方希望立即拿到一个 Stream。

Pi 使用 lazyStream：

~~~text
调用 streamSimple()
    ↓
立即返回外层 Stream
    ↓
异步完成认证和 Adapter 加载
    ↓
连接内层 Stream
    ↓
转发所有事件
~~~

如果初始化阶段失败：

~~~text
认证失败 / Adapter 加载失败
        ↓
创建 AssistantMessage
stopReason = error
        ↓
发送统一 error event
~~~

这样初始化错误和请求过程中的错误都进入同一个事件协议，不需要 Agent Loop 维护两套异常路径。

## 13. Harness 与 Adapter 的边界

这一部分虽然更接近底层基础设施，但仍然属于 Harness 的关键组件。原因是：Agent Loop 只能通过它把抽象的 Context 变成真实的模型请求，并把模型响应重新变成 Agent 能理解的事件。

~~~text
Agent Loop
    │  Context + Model + streamSimple()
    ↓
ModelRuntime
    │  模型选择、认证解析、请求准备
    ↓
Provider
    │  Provider / Model / API 实现选择
    ↓
API Adapter
    │  请求协议转换、流式响应解析
    ↓
模型服务
    │
    ↑ 统一的 AssistantMessage / StreamEvent
~~~

Agent Core 负责：

- 组装消息和工具上下文；
- 调用模型；
- 判断是否产生 Tool Call；
- 执行工具并继续或结束 Loop；
- 处理 Steering、Follow-up、Compaction 等运行控制。

Model Runtime 和 Adapter 负责：

- 找到可用的 Model；
- 解析 Provider 和认证；
- 选择正确的 API 协议；
- 把通用消息转换成 Provider 请求；
- 解析响应并发出统一事件；
- 把底层错误归一化为 Agent 可处理的错误。

因此，Adapter 不决定“模型下一步做什么”。它只负责“如何调用这个模型，以及如何把结果交回来”。

## 14. 设计取舍

### 14.1 好处

- Agent Core 不需要知道 Anthropic、OpenAI、Google 等协议细节；
- 所有 Provider 都通过同一套流式事件进入 Agent Loop；
- 认证、Model 列表和请求配置可以集中管理；
- 同一个 Agent Loop 可以复用不同 Provider；
- 新增 Provider 时，主要增加 Model 描述和 API Adapter，不需要重写 Agent Loop。

### 14.2 必要的复杂度

- 不同协议的消息结构、工具格式和推理参数不同，需要 Adapter 转换；
- Provider 的流式事件不一致，需要统一成 start、delta、toolcall、done、error 等事件；
- Model 元数据必须准确，否则模型选择、Context Window 或推理参数可能错误；
- Provider 配置、认证覆盖、Extension Provider 和兼容层会增加组合复杂度；
- 协议升级可能要求同时维护多个 Adapter 和兼容逻辑。

这里的核心取舍是：

~~~text
把 Provider 差异放在底层集中处理
        ↓
换取 Agent Loop 的稳定和简单
        ↓
代价是 Model Runtime / Adapter 层更复杂
~~~

这不是把复杂度消除，而是把复杂度隔离在模型调用边界内。

## 15. 本章结论

Pi 的模型调用层可以抽象为五个职责：

~~~text
Model Descriptor
    → 描述“调用谁、能力是什么、限制是什么”

Model Resolver / ModelRuntime
    → 决定“当前能用哪个模型，并准备请求”

Provider
    → 组织“Provider、认证、模型和 API 实现”

API Adapter
    → 处理“具体协议、参数转换和响应解析”

Stream Normalizer
    → 把不同协议统一成 Agent Loop 的事件
~~~

最终对 Agent Loop 暴露的是一个稳定接口：

~~~text
Context + Model
    → streamSimple()
    → 统一的流式事件
~~~

所以 Agent Loop 不需要关心请求是 HTTP、SSE、Anthropic Messages 还是 OpenAI Responses；它只处理统一的模型消息和事件。

## 16. 关键源码定位

- Model、Api、Context、SimpleStreamOptions 和 StreamEvent：packages/ai/src/types.ts:17-29、313-322、521-551、820-850
- Provider 与 Models 抽象：packages/ai/src/models.ts:88-223
- Provider 创建、API 选择和分发：packages/ai/src/models.ts:739-861
- ModelRuntime 创建、Provider 组合和快照：packages/coding-agent/src/core/model-runtime.ts:129-314
- 认证解析、请求准备和 streamSimple：packages/coding-agent/src/core/model-runtime.ts:573-645
- Agent Session 创建和 streamFn 注入：packages/coding-agent/src/core/sdk.ts:173-217、306-372
- Agent Loop 构造 LLM Context 并调用 streamFunction：packages/agent/src/agent-loop.ts:277-312
- Lazy Stream 和初始化错误统一：packages/ai/src/api/lazy.ts:41-97
- OpenAI Completions Adapter：packages/ai/src/api/openai-completions.ts:281-329、683-702
- Anthropic Messages 的 streamSimple 映射：packages/ai/src/api/anthropic-messages.ts:825-871
- OpenAI Responses Adapter：packages/ai/src/api/openai-responses.ts:103-216
- 旧版兼容入口：packages/ai/src/compat.ts:1-10
