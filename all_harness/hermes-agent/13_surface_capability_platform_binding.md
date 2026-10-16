# Hermes Agent 13：Surface Capability / Platform Binding

> 状态：已共同阅读并确认。  
> 本章重点：同一套 Agent Core 如何根据当前 session 的接入面暴露不同能力，以及 Hermes 如何处理跨平台恢复与 prefix cache 之间的冲突。  
> 代码复核：基于 2026-09-17 当前 checkout。

## 1. 这一层解决的是什么问题

Hermes 的 Agent Core 同时服务于 CLI、TUI、Desktop、Telegram、Discord、Slack 等入口，但不同客户端真正能承载的能力并不相同：

~~~text
Desktop
  ├── project
  ├── desktop_ui
  ├── pane / preview / in-app browser
  └── 通用工具

TUI
  ├── project
  └── 通用工具

Telegram / Discord / Slack
  ├── 各自的平台能力
  └── 通用工具
~~~

因此，进程里“注册了哪些工具”和当前 Agent“实际向模型暴露哪些工具”不能是同一件事：

~~~text
Process Registry
  所有候选能力
        │
        ▼
Surface / Capability Resolver
  session source + operator config + reachability
        │
        ▼
Agent Tool Snapshot
  当前 Agent 真正发给模型的 tools schema
~~~

这一层的核心职责是：

> 根据当前 session 的真实接入面，建立一份正确且尽量稳定的 capability snapshot。

## 2. Surface 是 Session 属性，不是 Process 属性

Hermes 最值得记住的约束是：

> 只有“连接另一端是谁”才能决定的能力，必须根据 session source/platform 解析，不能只看后端进程环境变量。

例如 Desktop 可以连接：

~~~text
本地后端
SSH 后端
URL 后端
Hermes Cloud
~~~

只有部分后端是由 Electron 启动的，因此 `HERMES_DESKTOP=1` 只能说明：

~~~text
这个后端进程由 Desktop 启动
~~~

它不能可靠说明：

~~~text
当前这条 session 的另一端一定是 Desktop UI
~~~

Hermes 因而把 `desktop_ui`、`project` 放在单独 toolset 中，由 session platform 决定是否折入当前 Agent 的 enabled toolsets：

~~~text
platform = desktop
  → project + desktop_ui

platform = tui
  → project

其他消息平台
  → 不承担这些 GUI schema
~~~

`check_fn` 也不适合承担 surface 判断。它更适合回答：

- 服务是否配置；
- bridge 是否可达；
- 用户是否开启某能力。

原因是 `check_fn` 结果可能进程级缓存，而同一后端进程可能同时服务多个不同来源的 session。

## 3. Agent Tool Snapshot 在哪里形成

TUI/Desktop 路径中，`session.create` 或 `session.resume` 会把 source/platform 传给 `_make_agent()`：

~~~text
session source
      │
      ▼
_resolve_agent_platform()
      │
      ▼
_load_enabled_toolsets(platform)
      │
      ▼
AIAgent(enabled_toolsets=..., platform=...)
      │
      ▼
agent.tools
~~~

关键代码边界：

- `toolsets.py`：声明工具组；
- `tui_gateway/server.py::_gui_surface_toolsets()`：把客户端 surface 映射成 toolsets；
- `tui_gateway/server.py::_load_enabled_toolsets()`：合并 coding posture、配置、MCP 和 surface toolsets；
- `tui_gateway/server.py::_make_agent()`：用解析后的 platform/toolsets 创建 AIAgent；
- `model_tools.py::get_tool_definitions()`：把注册表筛选成实际工具 schema；
- `agent/agent_init.py`：形成当前 Agent 的工具快照。

`desktop_ui` 和 `project` 刻意不在 `_HERMES_CORE_TOOLS` 中。否则所有 Telegram、CLI、Cron 等请求都要为根本不可用的 GUI schema 付 token 成本。

## 4. 工具快照“稳定”到底是什么意思

不能把它概括成：

> 一个持久化 session 创建后，工具集永远不允许变化。

更准确的是：

> 一个活跃 AIAgent 实例在建立时取得工具快照，之后不会在每次 API call 前重新读取注册表；同一接入面内应尽量维持这份快照稳定。

因此：

~~~text
同平台、同一个活跃 Agent 生命周期
  → system prompt / tools 尽量稳定
  → prefix cache 友好

重新创建或显式跨 surface 恢复 Agent
  → 可以根据新的 session source 重新解析 capability
~~~

这个区别对理解跨平台恢复非常重要。

## 5. 普通跨平台聊天：默认不是同一个 Session

Gateway 的确定性 session key 包含平台：

~~~text
agent:main:telegram:dm:...
agent:main:discord:dm:...
agent:main:slack:dm:...
~~~

所以，同一个用户从 Desktop 和 Telegram 聊天时，默认得到的是两个逻辑 session：

~~~text
Desktop Session
  ├── desktop system prompt
  ├── desktop tools
  └── desktop cache scope

Telegram Session
  ├── telegram system prompt
  ├── telegram tools
  └── telegram cache scope
~~~

Hermes 保持 prefix cache 稳定的主要办法并不是“跨平台换了工具也能继续命中”，而是：

> 默认不要把不同平台自动压成同一个运行中会话；让每个平台在自己的 session 内保持稳定。

## 6. 显式恢复同一个 Session：能力正确性优先

Hermes 也允许某个客户端显式打开已有持久化 session。例如 Desktop 可以查看并继续一个原本来自 Telegram 的会话。

Desktop 调用 `session.resume` 时会明确传入：

~~~json
{
  "session_id": "原 Telegram session",
  "source": "desktop"
}
~~~

恢复过程是：

~~~text
原 Telegram Session
  ├── 历史 messages
  ├── Telegram system prompt
  └── Telegram tool snapshot
              │
              │ Desktop: session.resume(source="desktop")
              ▼
读取同一份持久化 transcript
              │
              ▼
按 desktop 重新构造 AIAgent
  ├── platform = desktop
  ├── enabled toolsets += project + desktop_ui
  ├── 形成新的 agent.tools snapshot
  └── 校验旧 system prompt 的 runtime identity
~~~

这里继承的是历史 conversation，不是原平台永远冻结的能力。

旧的 tool calls 和 tool results 仍然作为历史证据留在 transcript 中；新的 tool snapshot 只决定模型从现在开始还能调用什么。

## 7. System Prompt 也会检查平台漂移

Hermes 会优先从 SessionDB 恢复原 system prompt，以保持字节级稳定。但它不是无条件复用，而会检查：

~~~text
Model
Provider
Current working directory
Platform
~~~

如果原 prompt 是：

~~~text
Platform: telegram
~~~

而当前 AIAgent 是：

~~~text
Platform: desktop
~~~

`_stored_prompt_matches_runtime()` 会拒绝旧 prompt，并重新构建、重新持久化 system prompt。

这避免了：

~~~text
模型拿到了 Desktop 工具
但 system prompt 仍告诉它当前位于 Telegram
~~~

因此跨 surface 恢复不仅可能改变 tools，也可能改变 system prompt。

## 8. Prefix Cache：不会强行保住完整命中

Hermes 的 `prompt_cache_key` 不是简单等于 `session_id`。支持该能力的 transport 会把静态 instructions、当前 tools 和 logical session scope 一起参与计算：

~~~text
prompt_cache_key = hash(
    static instructions
    + current tools
    + logical session scope
)
~~~

所以即使 session scope 不变：

~~~text
hash(prompt + Telegram Tools + same scope)
                      ≠
hash(prompt + Desktop Tools + same scope)
~~~

对于使用严格 prefix matching / cache-control 的 Provider，道理也相同：system prompt 或 tools 的字节发生变化后，不能再把旧请求当成完整相同前缀。具体还能复用哪些更早的稳定块取决于 Provider 和 wire layout，但不能保证长历史继续完整命中。

完整决策可以画成：

~~~text
                  用户从另一个平台进入
                            │
                            ▼
                是否自动映射为同一 session？
                     /                 \
                   否                   是，显式 resume
                   │                         │
           创建平台独立 session              ▼
           独立 tools / cache       读取原来的历史 messages
                                             │
                                             ▼
                               按当前平台重建 AIAgent
                               ├── 重新选择 toolsets
                               ├── 创建新的 tool snapshot
                               └── 检查 system prompt
                                             │
                                   Platform 是否相同？
                                      /             \
                                    是               否
                                    │                │
                             复用原 system      重建 system prompt
                                    │                │
                                    └──────┬─────────┘
                                           ▼
                               instructions/tools 是否变化？
                                      /             \
                                    没变             变化
                                    │                │
                             cache 可继续命中    cache key 改变 /
                                               exact prefix 失配
~~~

因此 Hermes 的真实优先级是：

~~~text
当前 surface 的能力与提示正确
            >
保留旧工具 schema
            >
完整 prefix cache 命中
~~~

它不会为了缓存而：

- 继续暴露当前客户端无法承载的工具；
- 把所有平台工具做成永久大并集；
- 让所有请求承担所有 surface schema 的 token 成本；
- 让 prompt 中的平台说明与真实 UI 不一致。

显式 surface migration 产生的一次 cache break，是 Hermes 为正确性主动接受的代价。

## 9. 对 AI-Human Runtime 的启发

这套设计可以抽象为：

~~~text
Capability Registry
        │
        ▼
Session Capability Resolver
  app / surface / tenant / feature flags / reachability
        │
        ▼
Agent Capability Snapshot
        │
        ├── model-visible tool schemas
        ├── runtime prompt projection
        └── event / UI capability
~~~

值得直接保留的规则：

1. Process Registry 只表示候选能力，不等于每个 Agent 都应该看到。
2. UI 能力由 session/client identity 决定，不由后端进程启动方式决定。
3. Reachability、用户开关和 surface identity 是三个不同维度，不要都塞进 `check_fn`。
4. 一个活跃 Agent 内的 capability snapshot 应保持稳定，避免逐轮漂移。
5. 如果显式迁移到新 surface，先保证 prompt 和工具能力正确，再接受必要的缓存失效。
6. 历史 tool result 与当前可调用工具应分开：前者是 transcript 证据，后者是当前 capability。
7. 缓存键应包含真正影响模型输入的 instructions/tools，不能只依赖 session_id 强行复用。

一句话总结：

> Hermes 默认用平台级 session 隔离维持缓存稳定；当用户显式跨 surface 恢复同一会话时，它保留历史，但按当前客户端重新绑定 system prompt 和工具能力，并把必要的 cache break 视为正确性成本。
