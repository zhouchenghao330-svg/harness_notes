# zvec-grep：Agent Workspace Retrieval 拆解

> 状态：第一轮 Agent-centric 主链、索引、Runtime 和 retrieval 已讨论；外围模块暂不展开
>
> 研究边界：只研究 zvec-grep 如何影响 Agent 获取、选择和消费本地证据；不做完整产品架构导览。

## 1. 研究定位

zvec-grep 不是完整的 Agent Harness，而是 Harness 可以调用的本地 Workspace Retrieval / Evidence Tool。

它不负责：

- 模型调用
- Agent 主循环
- Session、Memory、Compaction
- 通用 Tool Loop

它负责：

- 引导 Agent 选择语义检索或精确检索
- 将工作区内容建立为可检索的结构化索引
- 对查询进行 FTS + Vector 混合召回和排序
- 将结果压缩为带路径、行号和源码片段的 Agent Context
- 处理索引新鲜度、后台刷新和远程 Embedding 授权

## 2. Agent-centric 主路径

```text
Agent
  ↓ MCP tools/call
zvec_grep_search
  ↓ 输入规范化、检索分组
DaemonBackend.search
  ↓ 工作区运行时、新鲜度、模型
RootRuntime.search
  ↓ 只读索引会话
ZvecGrepService.context
  ↓
WorkspaceIndex.searchPlan
  ↓ FTS / Vector / RRF
紧凑的本地证据
  ↓
Agent 继续推理
```

## 3. 研究章节

- [[01_scope_and_boundary]]
- [[02_agent_integration_and_mcp]]
- [[03_indexing_and_evidence]]
- [[04_search_pipeline]]
- [[05_runtime_freshness_and_authorization]]
- [[06_agent_patterns_and_open_questions]]

## 4. 当前第一轮结论

- `zg install` 不只是写入 MCP 地址，还会向 Agent 配置写入检索路由指导。
- `zvec_grep_search` 面向语义、关系、架构和跨文件问题；精确字符串和正则仍然走 `rg`。
- `query` 默认形成一个 FTS + Vector 混合检索组。
- 搜索结果是有界的 ranked sample，不是完整文件读取。
- 默认搜索可能触发后台索引刷新，因此“读搜索”存在运行时副作用。
- 默认 `agent` MCP toolset 只暴露 `zvec_grep_search`；索引维护和 exhaustive rg 被放到 full toolset。
- 搜索响应把 freshness、background refresh、路径/范围和 bounded snippet 一起交给 Agent。
- 没有索引时不能静默创建持久状态；远程 Embedding 未获授权时可以改走本地 FTS 或停止。
- retrieval 是两层 rank-based RRF：先融合 FTS/Vector，再融合多个 query groups；没有 cross-encoder 或 LLM reranker。
- watcher 提供的是带 debounce 的后台最终一致性，不是逐字符实时索引。
- 对代码精确搜索和穷举分析，`rg`/LSP 更合适；zvec-grep 的窄价值是未知关键词时的 Agent 语义发现。

## 6. 端到端主链（第一轮核对）

```text
MCP tools/call
  → normalizeSearchInput
  → remote authorization branch（仅远程 Embedding）
  → DaemonBackend.search
  → RootRuntime（watcher / model pool / read session）
  → ZvecGrepService.context
  → WorkspaceIndex.searchPlan
  → compact Agent context + freshness/indexing state
```

代码入口：`src/mcp/tools.ts:434-510`、`src/mcp/input-normalization.ts:40-157`、`src/daemon/backend.ts:432-585`、`src/daemon/root-runtime.ts:97-151`。

## 5. 证据标记

- **Code-verified**：代码已确认
- **Docs-described**：文档或工具说明这样描述
- **Inferred**：根据代码行为推断
