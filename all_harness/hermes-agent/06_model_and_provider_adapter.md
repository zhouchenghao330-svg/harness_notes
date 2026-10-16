# Hermes Agent 06：Model / Provider / Transport

> 状态：已共同阅读并确认。  
> 本章重点：ProviderProfile 如何把 Provider 差异组织成可复用的声明式契约。

## 1. 核心结论

Hermes 的 06 负责把：

~~~text
用户选择的模型
    ↓
provider + api_mode + base_url + credentials
    ↓
协议请求
    ↓
Provider 原生响应
    ↓
统一 NormalizedResponse
~~~

它不负责 Agent 的决策，也不负责判断“下一步要不要调用工具”。

~~~text
Provider / Transport
  负责：如何调用模型、如何解析模型响应

Agent Loop
  负责：调用工具、继续、重试、nudge、结束
~~~

Hermes 的模型调用层可以拆成：

~~~text
模型选择和运行时解析
  hermes_cli/runtime_provider.py
  hermes_cli/model_switch.py

Provider 身份和能力描述
  providers/ProviderProfile
  hermes_cli/providers.py

协议转换
  agent/transports/

客户端、流式生命周期、凭证刷新
  AIAgent / agent/chat_completion_helpers.py

Agent 级别的重试和恢复
  agent/conversation_loop.py
~~~

## 2. Provider 和 api_mode 是两个维度

~~~text
provider
  = 调用谁、使用什么凭证、默认地址和特殊配置

api_mode
  = 按什么线协议发送请求
~~~

因此，一个 Provider 可以支持多个协议：

~~~text
provider = nous
model = anthropic/claude-sonnet
api_mode = anthropic_messages

provider = nous
model = deepseek/deepseek-v4
api_mode = chat_completions
~~~

多个 Provider 也可以共用同一个协议：

~~~text
chat_completions
  ├── OpenRouter
  ├── DeepSeek
  ├── Qwen
  ├── Kimi
  ├── Ollama
  └── 自定义 OpenAI-compatible Provider
~~~

协议通常由以下信息共同决定：

1. 用户显式配置的 api_mode；
2. endpoint 的 host/path；
3. Provider 的默认声明；
4. 具体模型的协议要求；
5. 对未知自定义端点使用 chat_completions 作为保守默认。

切换模型时要依据目标模型重新计算 api_mode，不能把上一个模型的协议状态直接带过去。

## 3. ProviderProfile 是 Provider Contract

ProviderProfile 不只是配置对象，而是一份 Provider 在整个系统中的能力和行为合同。

~~~text
ProviderProfile
├── name / aliases
├── api_mode
├── env_vars / auth_type
├── base_url / models_url
├── supports_vision
├── supports_vision_tool_messages
├── supports_prompt_cache_key
├── default_headers
├── default_max_tokens
├── prepare_messages()
├── build_extra_body()
└── build_api_kwargs_extras()
~~~

它描述的问题包括：

~~~text
这个 Provider 如何认证？
默认请求发到哪里？
模型列表如何获取？
是否支持视觉输入？
是否允许 prompt_cache_key？
reasoning 参数放在哪里？
需要哪些额外 header？
max_tokens 有什么默认值？
消息发送前需要做什么预处理？
~~~

重要的是：Profile 不负责创建客户端、发起请求、重试、中断或 Agent Loop。

~~~text
ProviderProfile
  = 声明式配置 + 少量请求变换 Hook

ProviderTransport
  = 协议结构转换

AIAgent
  = 客户端和调用生命周期

Agent Loop
  = 执行控制和恢复
~~~

## 4. 最优秀的设计：把差异分类

不是所有差异都用同一种方式解决。

| 差异类型 | 解决方式 |
| --- | --- |
| 稳定的 Provider 数据 | Profile 字段 |
| 同协议下的局部行为差异 | Profile Hook |
| 完全不同的消息协议 | 新增 Transport |
| 模型自身的 context/cost/vision 能力 | Model Metadata |
| 当前请求的临时覆盖 | request_overrides |
| 重试、凭证刷新、中断 | AIAgent / Agent Loop |

可以记成：

~~~text
静态差异       → Profile 字段
局部行为差异   → Profile Hook
协议结构差异   → Transport
模型级能力     → Model Metadata
运行时状态     → AIAgent
执行控制       → Agent Loop
~~~

这样既不会把所有 Provider 差异写成 if/elif，也不会把 ProviderProfile 变成一个巨大的 God Object。

## 5. Hook 设计比字段数量更重要

其中最值得记忆的是：

~~~python
build_api_kwargs_extras()
    → (extra_body_additions, top_level_kwargs)
~~~

因为不同 Provider 可能表达同一个语义，但字段位置不同：

~~~text
reasoning_effort
  Provider A → 顶层字段

reasoning
  Provider B → extra_body.reasoning

thinking_budget
  Provider C → extra_body.thinking_budget
~~~

核心逻辑只表达：

~~~text
开启 reasoning
~~~

ProviderProfile 再决定最终 wire payload：

~~~text
语义配置
    ↓
ProviderProfile
    ↓
字段名称和字段位置
~~~

这是一种“语义和线协议位置分离”的设计。

## 6. Capability 要细粒度，而且默认保守

Hermes 没有只使用一个粗糙的 supports_vision，还区分：

~~~text
supports_vision
supports_vision_tool_messages
supports_prompt_cache_key
~~~

真实世界里可能出现：

~~~text
Provider 支持图片输入
但不支持 tool result 中的图片

Provider 支持 Chat Completions
但不接受 prompt_cache_key

Provider 支持 reasoning
但不接受某些 reasoning_effort 值
~~~

尤其值得记忆：

~~~python
supports_prompt_cache_key = False
~~~

未知能力默认关闭，而不是假设所有 OpenAI-compatible 服务都会忽略未知字段。

这属于：

> 能力必须显式声明，默认采取保守行为。

同时，Provider 级能力和 Model 级能力要分开：

~~~text
ProviderProfile
  描述整个服务的稳定能力和协议习惯

Model Metadata
  描述某一个模型的 context window、价格、vision、
  reasoning、tool calling 等能力
~~~

## 7. Profile 是全系统共享契约

ProviderProfile 不只被 Transport 使用。它还可以被：

~~~text
auth
model catalog
model picker
setup wizard
doctor health check
runtime provider resolver
model metadata
auxiliary client
chat transport
~~~

这样可以避免每个模块维护自己的 Provider 判断：

~~~text
auth.py       一套 Provider 信息
models.py     一套 Provider 信息
doctor.py     一套 Provider 信息
transport.py  一套 Provider 信息
~~~

否则很容易出现：

~~~text
模型选择界面说支持
实际请求却 400

setup 找得到 Provider
runtime 却不知道如何调用

doctor 检查通过
真正请求却缺少必要 header
~~~

所以 Profile 的价值不是“少写几个 if”，而是建立了跨模块的一致身份和能力合同。

## 8. Hook 的调用顺序也是契约

例如消息预处理不是随便什么时候执行：

~~~text
清理 Hermes / Codex 内部字段
        ↓
Provider.prepare_messages()
        ↓
通用 role 转换
        ↓
构造最终 payload
~~~

Hook 设计需要同时规定：

~~~text
什么时候调用？
输入处于什么状态？
允许修改哪些字段？
后续还会经过哪些转换？
~~~

只有明确生命周期，插件才能安全扩展；否则不同 Hook 之间会互相覆盖、破坏缓存或把内部字段泄露到 Provider。

## 9. Profile + Plugin 让核心保持窄

理想情况下，新增 Provider 只需要：

~~~text
plugins/model-providers/acme/
        ↓
注册 ProviderProfile
        ↓
自动进入：
  auth
  model picker
  setup
  doctor
  runtime
  transport
~~~

而不需要改动：

~~~text
run_agent.py
conversation_loop.py
model_tools.py
cli.py
~~~

这符合 Hermes 的整体原则：

> 核心只提供通用能力，Provider 差异放在边缘插件。

Hermes 当前仍保留 legacy flag path，说明 ProviderProfile 体系处于逐步迁移中：已注册 Provider 走 Profile path，未知 Provider 仍可走旧的兼容逻辑。这是一个值得学习的迁移方式：

~~~text
先建���新抽象
        ↓
新 Provider 走新抽象
        ↓
旧 Provider 保留兼容路径
        ↓
逐步迁移，而不是一次性重写
~~~

## 10. Transport 的职责边界

ProviderTransport 的统一接口大致是：

~~~text
convert_messages()
convert_tools()
build_kwargs()
normalize_response()
validate_response()
extract_cache_stats()
map_finish_reason()
~~~

它负责：

~~~text
统一内部消息
    ↓
Provider 原生请求
    ↓
Provider 原生响应
    ↓
统一 NormalizedResponse
~~~

它不负责：

- 创建客户端；
- 凭证刷新；
- 重试；
- 中断；
- prompt cache 生命周期；
- Agent Loop。

这使得同一个 Agent Loop 可以复用多个 Provider 和协议。

## 11. NormalizedResponse 的设计

不同 Provider 的公共语义放在统一字段：

~~~python
NormalizedResponse(
    content,
    tool_calls,
    finish_reason,
    reasoning,
    usage,
    provider_data,
)
~~~

协议专属字段放进 provider_data，例如：

~~~text
Gemini:
  tool_call.provider_data["extra_content"]
  └── thought_signature

Anthropic:
  reasoning_details
  anthropic_content_blocks

Codex:
  codex_reasoning_items
  codex_message_items
  call_id
  response_item_id
~~~

这避免了两个极端：

~~~text
极端一：Agent Loop 到处判断 Provider

极端二：为了统一而丢弃 replay 所需的协议元数据
~~~

公共接口保持稳定，协议恢复所需的信息仍然被保留。

## 12. 和 thinking 恢复的关系

Model / Provider / Transport 负责：

~~~text
可靠地观察和翻译底层模型发生了什么
~~~

Repair 层负责：

~~~text
判断这个响应是否构成合法 Agent Turn
决定 retry、continuation、nudge 或丢弃
~~~

例如：

~~~text
stream 没有 terminal event
        ↓
Transport / stream runtime 标记 invalid 或 incomplete
        ↓
Agent Loop retry / fallback

reasoning 有内容，但 content 为空
        ↓
NormalizedResponse 正常交回
        ↓
02.5 repair 决定 nudge / replay / fresh retry
~~~

因此，nudge 不应该塞进 Transport。它是 Agent Loop 的语义恢复动作，不是协议转换动作。

## 13. 最值得记忆的抽象

~~~text
ProviderProfile
  不是 Adapter

ProviderProfile
  是给通用 Adapter 提供能力描述和局部行为注入的 Provider Contract
~~~

再压缩成一句：

> 用 Profile 描述“这个 Provider 有什么稳定差异”，用 Hook 注入“这个 Provider 在请求构造时有什么局部差异”，用 Transport 处理“协议本身如何转换”，把运行时控制留给 Agent Loop。

## 14. 关键源码

- ProviderProfile：providers/base.py
- Provider Profile 注册和插件发现：providers/__init__.py
- Profile 如何被全系统消费：providers/README.md
- Provider 和 api_mode 解析：hermes_cli/runtime_provider.py
- Provider / endpoint 的协议判断：hermes_cli/providers.py
- Agent 初始化和客户端路由：agent/agent_init.py
- Transport 抽象：agent/transports/base.py
- Transport 注册：agent/transports/__init__.py
- 统一响应类型：agent/transports/types.py
- Chat Completions 请求构造：agent/transports/chat_completions.py
- Responses API 转换：agent/transports/codex.py、agent/codex_responses_adapter.py
- API 调用和流式生命周期：agent/chat_completion_helpers.py
- Agent 级别的响应归一化、重试和恢复：agent/conversation_loop.py
