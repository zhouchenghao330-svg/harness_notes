# Hermes Agent 04：Context、Prompt 与 Prompt Cache

> 状态：已共同阅读并确认。本文关注 Hermes 如何在加入 Memory、Skill、Project Context、Plugin 和临时恢复信息的同时，保持模型请求前缀稳定。
>
> 研究日期：2026-08-31

## 1. 核心问题

04 不只是“怎样拼一段 Prompt”。它解决的是一个直接影响成本、延迟与行为一致性的矛盾：

~~~text
模型需要看到越来越多的动态信息
        vs
动态信息越多，Prompt Cache 越容易失效
~~~

Hermes 的总体策略可以概括为：

~~~text
稳定内容尽量靠前
易变内容尽量靠后
真正临时的内容不要改长期 System Prompt
请求前统一规范化
规范化结束后，最后才添加 Provider Cache Marker
~~~

因此，Prompt 是一套状态边界和缓存边界设计，而不是单纯的字符串拼接。

## 2. 先分清三种“缓存 / 保存”

| 名称 | 它保存什么 | 目的 |
|---|---|---|
| `_cached_system_prompt` | 已组装好的系统提示词字符串 | Hermes 自己不在每一轮重新拼 System Prompt |
| Provider Prompt Cache | Provider 侧已计算过的前缀 KV | 减少输入 token 和延迟 |
| `api_content` | 某条历史消息当时真正送到 API 的内容 | 恢复时按原内容重放，保持前缀一致 |

`_cached_system_prompt` 不是 Provider Cache；它只是 Hermes 保证“本会话一直使用同一份 Prompt 文本”的本地状态。真正的 Provider Cache 由 Anthropic 的 `cache_control` 或 Responses 的 `prompt_cache_key` 等协议能力承载。

## 3. 为什么需要 `api_content`？

假设用户输入是：

~~~text
帮我看这个 bug
~~~

本轮模型请求前，Hermes 可能附加：

~~~text
[Memory recall]
用户正在维护 hermes-agent。

[Plugin context]
当前工作区是 ...
~~~

用户可见的聊天记录不能因此变成一条包含内部召回信息的 user message；但下一轮若只重放干净文本，模型历史就与第一次实际请求不同，Provider 的 Prefix Cache 会从这里断掉。

所以 Hermes 保存两份语义：

~~~text
content
  = 用户可见的干净文本

api_content
  = 当时真实发送给模型的版本
  = 原问题 + recall + plugin context
~~~

这不是业务 Memory，而是“模型输入历史的回放 sidecar”。

## 4. System Prompt 的三层

`build_system_prompt_parts()` 把 System Prompt 组织为三段：

~~~text
System Prompt
├─ stable
├─ context
└─ volatile
~~~

### 4.1 stable

跨 session 也尽量稳定的长前缀，例如：

- SOUL.md 或默认身份；
- 通用的完成任务、避免伪造、工具调用、并行工具调用规范；
- 与当前工具集稳定对应的工具行为指导；
- coding posture 与稳定环境提示；
- 固定 model/provider 的特殊指令。

这一段适合放最前面，争取跨 session 也能被最长前缀缓存复用。

### 4.2 context

工作区或 session 相对稳定、但不应与跨项目身份混在一起的内容，例如：

- coding workspace snapshot；
- 项目 Context 文件；
- platform hint；
- caller 传入的 `system_message`。

项目 Context 文件按优先级选择，而不是把所有 Agent 生态的规则全堆进去：

~~~text
.hermes.md / HERMES.md
  ↓ 没有才看
AGENTS.md（git root → cwd 的目录链）
  ↓ 没有才看
CLAUDE.md
  ↓ 没有才看
.cursorrules / .cursor/rules
~~~

`SOUL.md` 独立作为身份来源。每种项目规则都有长度上限；AGENTS.md 允许从仓库根向当前目录逐层合并，更深的规则在后面，语义更具体。

### 4.3 volatile

最可能在“下次重建 Prompt 时”变化的内容，例如：

- skill index；
- 内置 Memory / USER profile；
- 外部 memory provider 的系统级块；
- plugin prompt section；
- 日期、时区、model/provider/platform 信息。

这里的 `volatile` 不表示它每次 API call 都会变。相反，完整 System Prompt 一旦建立，会被冻结在 `_cached_system_prompt`，本 session 内尽量原样重放。`volatile` 的含义是：如果压缩、恢复或显式失效导致必须重建，最可能变化的内容被放到最后，从而让前面的 stable/context 前缀仍有机会复用。

~~~text
旧 Prompt: [stable][context][volatile-old]
新 Prompt: [stable][context][volatile-new]
                    ↑
          前面的 KV 仍可能复用
~~~

## 5. 临时 Context 应该放在哪里？

这条是 Hermes 最值得复用的规则：

~~~text
长期、会话级规则
  → System Prompt

本轮 recall
pre_llm_call plugin 输出
gateway 临时说明
nudge / continuation
  → 当前 user message 的 API request copy
~~~

原因很直接：若每轮把 recall 和 plugin 输出拼进 System Prompt，系统提示词每轮变化，前缀缓存会持续失效。

因此当前 user message 通常是：

~~~text
稳定历史前缀
  + 当前用户问题
  + 本轮 recall / plugin context
~~~

它只增加本轮请求的尾部，不回写旧历史；同一 Turn 内多次 API call 则复用同一份已组成的内容。

## 6. 从 canonical history 到 Provider Request

一次 API 请求大致这样构造：

~~~text
canonical messages
  ↓ structural clone
api_messages
  ↓
恢复历史 api_content
注入当前 recall / plugin context
移除 UI、DB、display 私有字段
映射 reasoning 和清理 Provider 不认识的字段
  ↓
加入 cached system prompt
加入 ephemeral system prompt / prefill（若存在）
  ↓
ContextEngine.select_context()（可选，仅本请求）
  ↓
修复 orphan tool result / role sequence
drop thinking-only API copy
规范化空白和 tool-call JSON
  ↓
最后添加 Provider Cache Marker
  ↓
Provider request
~~~

关键原则：`messages` 是 canonical working history，`api_messages` 是可随 Provider 规则变换的请求副本。任何 Provider 清洗、thinking-only drop、cache marker、reasoning echo 都不应直接改写长期 transcript。

## 7. 为什么 Cache Marker 必须最后添加？

Anthropic 风格的 cache marker 往往会把普通字符串内容改为 content block 列表。若先打 marker、后做空白清理或 role/tool 修复，会出现同一条历史消息在不同轮次发出不同字节：

~~~text
第一次：tool result 末尾保留换行
下一次：同一条 tool result 被 strip
  ↓
Prefix 在这里断掉，缓存失效
~~~

因此 Hermes 的顺序是：

~~~text
所有 API message 变换完成
  ↓
内容标准化
  ↓
最后生成 request-local cache plan
~~~

cache marker 只存在于 API 副本；canonical tool registry 和持久化 transcript 不带 marker。

## 8. Anthropic 风格缓存计划

对启用 Anthropic Prompt Cache 的路径，默认最多使用四个 breakpoint：

~~~text
[static system prefix]
[full system prompt]
[最近一个完整交互边界]
[最近一个完整交互边界]
~~~

没有可验证的 static system prefix 时，会退化为：

~~~text
[full system prompt]
[最近三个可缓存消息 / 交互边界]
~~~

这里“完整交互边界”很重要：`assistant(tool_call) + tool result` 不能在中间随意断开。缓存点会优先落在完整 tool transaction 的结尾。

在部分原生 Anthropic 工具缓存能力下，Hermes 还可以把一个 cache breakpoint 给稳定的 tools schema；这是“工具面也属于长前缀”的直接体现。

TTL 可以是 `5m` 或 `1h`，但 Qwen/Alibaba 类路由会被强制收敛到其支持的 `5m`。

## 9. Responses 路径的 Cache Scope

Responses/Codex 类路径不一定使用 Anthropic 的 content block marker，而是可以传 `prompt_cache_key`。

这个 key 由稳定的 instructions、tools schema 和逻辑会话范围共同决定。逻辑范围不能机械使用物理 `session_id`：

~~~text
压缩导致 session rotation
  → 物理 id 改了
  → 逻辑会话仍是同一个
  → cache scope 应沿用 compression lineage root

/new、/branch、subagent
  → 真正的新会话或 fork
  → cache scope 应隔离
~~~

因此，压缩不会无意义地换一个缓存桶；但分支和子 Agent 不会意外共享上下文缓存范围。

## 10. 与“双存储” Agent 架构的对应

如果系统已经有 LangGraph Checkpointer 和用户可见 `messages` 表，最清晰的概念划分是三层：

~~~text
Visible transcript
  用户和 UI 真正看见什么

Runtime checkpoint
  Graph 如何继续跑：tool state、路由状态、内部消息、nudge、recall snapshot 等

Model-facing replay view
  历史上到底给模型发过什么，
  或怎样确定性重建同一份请求内容
~~~

Hermes 的 `content + api_content` 是把 visible transcript 和 model-facing replay view 放在同一条消息记录中的字段级分离；双表/双存储则把这个分离提升到存储层，语义通常更清楚。

但 Checkpointer “存了全部状态”不自动保证能稳定重放模型上下文。若第一次请求使用 `recall A`，恢复时重新检索到了 `recall B`，则行为与 Prefix Cache 都会从那条历史消息开始漂移。

可选策略：

~~~text
A. 保存当时真正的 model-facing message
   → 恢复时原字节重放，最稳定

B. 保存 recall snapshot / version / retrieval result
   → 用快照确定性重建

C. 恢复后重新 recall
   → 接受一次 cache miss，换取最新记忆
~~~

Nudge 也遵循同样边界：它可以存在于 Runtime Checkpoint 中以便 in-flight recovery，但不应进入用户可见 messages 表；一次 Turn 真正结束后，下一次正常用户运行不应再次 replay 已完成的内部 nudge。

## 11. 本篇结论

> Context 设计的关键不是“Prompt 写得多好”，而是每一段信息都要回答：它是否稳定？属于谁可见？是否需要持久化？恢复时能否按原样重放？它改变时会从哪一个缓存边界开始失效？

## 12. 代码入口

- `agent/system_prompt.py`：三层 System Prompt、构建、失效与恢复 static prefix。
- `agent/prompt_builder.py`：SOUL、Skills、项目 Context 文件发现与截断。
- `agent/turn_context.py`：Memory/Plugin Context 与 `api_content` sidecar 的生成。
- `agent/conversation_loop.py`：canonical messages → api_messages 的请求投影与最后的 cache decoration。
- `agent/prompt_caching.py`：Anthropic 风格 breakpoint 计划与 request-local marker。
- `agent/prompt_cache_scope.py`：compression rotation 下稳定的 logical cache scope。
