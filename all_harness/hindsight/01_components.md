# Hindsight 组件索引

> 状态：组件 01 已共同阅读并写入；组件 02 Retain 的既定三个分组第 1 至 12 项已共同阅读并写入，本轮清单已完成。组件 03 第一至三组已共同阅读并写入；下一步为第四组写入一致性，第四、五组待共同阅读。进度更新日期：2026-10-08。
>
> 阅读原则沿用 [[00_拆解共识|拆解共识]]：先在对话中逐条读代码，再写组件细节；关键结论注明 Code-verified、Docs-described 或 Inferred。

## 1. 拆解边界

Hindsight 要按两种运行链看：

1. **记忆链**：`Retain` 将输入变成 bank 中的记忆；`Recall` 从 bank 检索相关内容；后台归纳和心智模型刷新维护上层知识。
2. **Reflect Agent 链**：`Reflect` 读取 bank 配置，构造 Prompt 和工具，循环调用模型及只读检索工具，最后生成回答。

对照 Pi / Hermes 时，重点比较 Reflect 的 Prompt、模型调用、工具生命周期、运行控制和输出；Retain / Recall 作为它的记忆基础另行拆解。通用管理 UI、SDK 生成、发布部署不进入核心地图，除非代码改变模型输入、工具可见性、运行状态或记忆结果。

## 2. 组件地图

| 编号  | 组件                        | 首轮代码入口                                                                                               | 研究问题                                        | 状态          |
| --- | ------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------- | ----------- |
| 01  | 入口、Bank 和请求上下文            | `api/http.py`、`api/mcp.py`、`engine/memory_engine.py`、`models.py`                                     | 外部请求如何绑定 bank、权限和取消信号？                      | [[01_entry_bank_request_context|已共同阅读]] |
| 02  | Retain 写入流水线              | `engine/memory_engine.py`、`engine/retain/orchestrator.py`、`retain/fact_extraction.py`                | 文本如何切分、提取事实、建链接并写入？同步和异步路径如何分流？             | [[02_retain_pipeline|既定第 1 至 12 项已共同阅读并写入]] |
| 03  | 记忆表示与持久化                  | `engine/memories/`、`engine/db/`、`engine/retain/fact_storage.py`                                      | facts、entities、links、source chunks 如何表示和关联？ | [[03_memory_representation_persistence|第一至三组已共同阅读并写入；第四、五组待共同阅读]] |
| 04  | Recall 检索与结果预算            | `engine/memory_engine.py` 的 `recall_async()`、`engine/search/retrieval.py`、`fusion.py`、`reranking.py` | 多路召回怎样并发、融合、重排和裁剪？                          | 待共同阅读       |
| 05  | Reflect Agent Loop        | `engine/memory_engine.py` 的 `reflect_async()`、`engine/reflect/agent.py`                              | 一次请求怎样从模型 Tool Call 走到结果回填、继续或完成？           | 待共同阅读 |
| 06  | Reflect Context 和 Prompt  | `engine/reflect/prompts.py`、`agent.py`、`engine/directives/`                                          | bank profile、query、指令和历史结果怎样进入最终模型请求？       | 待共同阅读       |
| 07  | Reflect Tool System 与具体工具 | `engine/reflect/tools_schema.py`、`agent.py` 的 `_execute_tool()`、`tools.py`、`presentation.py`         | 工具何时可见、怎样执行、怎样把结果呈给模型？                      | 待共同阅读       |
| 08  | 模型和 Provider 适配           | `engine/llm_wrapper.py`、`engine/llm_interface.py`、`engine/providers/`                                | `call_with_tools()` 如何转请求、归一化响应和处理缓存？       | 待共同阅读       |
| 09  | 观测归纳与心智模型刷新               | `engine/consolidation/consolidator.py`、`engine/mental_model_refresh.py`、`engine/memory_engine.py`    | 事实何时变成 observation / mental model，何时重写？     | 待共同阅读       |
| 10  | 控制、后台任务与输出                | `engine/reflect/agent.py`、`cancellation.py`、`worker/`、`engine/reflect/models.py`、`api/http.py`       | 迭代/上下文预算、超时、取消、失败、trace 和最终响应怎样处理？          | 待共同阅读       |

表中的路径均相对 `hindsight-api-slim/hindsight_api/`。组件划分是首轮阅读计划，不等于组件机制已经确认。

## 3. Reflect 工具清单：仅确认名称

`engine/reflect/tools_schema.py` 的 `get_reflect_tools()` 可按 bank、请求选项和文档存储设置筛选工具。【Code-verified】具体参数、工具结果、权限与错误边界还未逐个确认。

| 工具 | 首轮作用 | 后续状态 |
| --- | --- | --- |
| `search_mental_models` / `read_mental_models` | 搜索心智模型，再读取完整内容 | 待共同阅读 |
| `search_observations` | 搜索归纳出的观测 | 待共同阅读 |
| `recall` | 搜索原始事实与经历 | 待共同阅读 |
| `expand` | 展开来源片段或文档上下文 | 待共同阅读 |
| `done` | 提交最终答案或结构化文档 | 待共同阅读 |

详细研究时另建 `07_tools/00_inventory.md` 与核心工具笔记；不能把这张名称表当作工具研究完成。

## 4. 当前阅读顺序

按上面的组件地图从 01 到 10 顺序共同阅读。01 入口已写入 [[01_entry_bank_request_context]]；02 Retain 的既定第 1 至 12 项已共同阅读并写入；03 已建立 [[03_memory_representation_persistence]]，第一至三组已共同阅读并写入，下一步为第四组写入一致性。

组件 02 已建立 [[02_retain_pipeline]]，记录整体流程与三个细节分组。第一组切块、第二组事实抽取，以及第三组第 9 项实体解析、第 10 项 Embedding 输入加工、第 11 项链接创建、第 12 项增量块复用均已共同阅读并写入。用户明确学到了实体解析和 Embedding 输入加工方式。本轮清单已逐项核对完成。组件 03 第一至三组已共同阅读并写入，第四、五组尚未完成。observation_scopes 和候选归纳的问答属于第二组补充，不代表组件 09 已完成。后续发现新的 Runtime 操作时仍须补入清单并主动说明。

## 5. 抽象与比较笔记

[[memory_抽象层|Memory 抽象层]] 保存从已读 Retain 和讨论中提炼的通用问题：留下什么、留成什么、怎样确认身份、如何找回、怎样处理变化。该页区分通用设计解释与 Hindsight 的已核对实现，后续随 Recall、Reflect 和归纳维护的共同阅读继续修正；不改变上方组件进度，也不代替 Runtime 操作清单。
