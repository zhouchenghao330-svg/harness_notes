# Hindsight 拆解总览

> 状态：总览保留首轮架构地图；组件 01、02 的本轮清单已共同阅读并写入；组件 03 第一至三组已共同阅读并写入，第四、五组待共同阅读；组件 04 至 10 待共同阅读。首轮研究日期：2026-09-29。进度更新日期：2026-10-08。
>
> 本目录沿用 [[00_拆解共识|Harness 拆解共识]]。Hindsight 的主体是 Agent 的长期记忆系统；其中 `Reflect` 自身含有模型与工具循环。因此先画出记忆链与 Reflect Agent，再判断哪些设计能与 Pi、Hermes 的 Harness 对照。

## 1. 一句话理解

把一个 `bank` 当作某个用户、Agent 或项目的独立记忆空间：`Retain` 写入并提取事实，`Recall` 检索事实，`Reflect` 让模型使用只读检索工具收集证据并形成回答。观测（observations）和心智模型（mental models）是事实之上的归纳层。

这一定位由仓库根目录 `README.md` 描述；三条操作的主入口可从 `hindsight-api-slim/hindsight_api/engine/memory_engine.py` 的 `retain_batch_async()`、`recall_async()`、`reflect_async()` 核对。【Docs-described + Code-verified】

## 2. 当前架构地图

~~~text
外部 Agent / SDK / HTTP / MCP / 集成
              │ bank_id + 输入
              ▼
        MemoryEngine / Bank 边界
              │
              ├─ Retain：文本/附件 → 提取事实/实体/链接 → 写入 bank
              │                                           │
              │                            归纳 observations；刷新 mental models
              │                                           │
              ├─ Recall：查询 → 多路检索 → 融合/重排 → 结果 │
              │                                           │
              └─ Reflect：问题 + bank 配置                │
                    ↓                                      │
               Prompt + 动态工具列表                       │
                    ↓                                      │
               模型 Tool Call                              │
                    ↓                                      │
               只读工具查询 bank ◀─────────────────────────┘
                    ↓
               结果写回本次 messages → 继续取证或 done/最终回答
~~~

这张图只表达首轮已找到的主边界；后台归纳与心智模型刷新各自的触发条件、数据写入时序，仍待逐条追踪。【Inferred，待验证】

## 3. 与 Pi / Hermes 的关键区别

- Pi / Hermes 主要研究通用 Agent 运行时：处理用户任务、可调用多类外部工具，并维护会话运行状态。
- Hindsight 主要提供**供这些 Agent 调用的记忆能力**。它不能整体等同于一个通用 Agent Harness。【Docs-described + Code-verified】
- `Reflect` 是最适合按 Harness 方法研究的局部运行时：在 `reflect/agent.py` 中构造消息和工具，通过 `LLMProvider.call_with_tools()` 循环调用模型，执行检索工具，将结果放回消息，再完成回答。【Code-verified】
- `Retain` 与 `Recall` 是 Reflect 的数据与工具基础；它们各自值得拆解，但不应硬套成 Agent Loop。【Code-verified】

## 4. 首轮源码锚点

| 问题 | 代码入口 | 首轮结论 |
| --- | --- | --- |
| 请求从哪里进来 | `hindsight_api/api/http.py`、`api/mcp.py` | HTTP、MCP 是入口；需要继续追踪请求上下文与 bank 约束 |
| 谁编排三条操作 | `engine/memory_engine.py` | `MemoryEngine` 暴露 retain、recall、reflect 方法 |
| 记忆怎么写入 | `engine/retain/orchestrator.py` | `retain_batch()` 是写入流水线核心入口 |
| 记忆怎么找回 | `engine/search/retrieval.py`、`engine/memory_engine.py` | 多路检索、融合与重排在这些位置展开 |
| Reflect 怎么循环 | `engine/reflect/agent.py` | `run_reflect_agent()` 调用 `_run_reflect_agent_inner()`；该函数内有迭代、工具调用、回填和完成控制 |
| 工具从哪里来 | `engine/reflect/tools_schema.py`、`tools.py` | Schema 与实际工具实现分开，后续必须建立具体工具清单 |
| 模型怎样调用 | `engine/llm_wrapper.py` | `LLMProvider.call_with_tools()` 是 Reflect 的模型调用接口 |

以上只确认入口与边界，不代表每条完整调用链已经共同读完。详见 [[01_components|组件索引]]。

## 5. 下一步

当前按 [[01_components|组件地图]] 从 01 到 10 顺序阅读。01 入口、Bank 和请求上下文已写入 [[01_entry_bank_request_context]]；02 Retain 的整体流程及三个分组第 1 至 12 项已共同阅读并写入 [[02_retain_pipeline]]；03 第一至三组已写入 [[03_memory_representation_persistence]]，下一步为第四组写入一致性，第四、五组待共同阅读。

[[memory_抽象层|Memory 抽象层]] 保存本轮讨论提炼的通用框架及其与 Hindsight 的对应关系。它用于组织后续理解，随共同阅读持续补充，不替代组件地图和具体代码拆解。
