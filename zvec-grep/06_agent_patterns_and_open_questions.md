# zvec-grep 06：Agent Pattern 与待确认问题

## 0. 先区分它是什么

`zvec-grep` 不拥有模型调用、消息历史、工具循环或最终回答。它是被 Harness 接入的外部 Retrieval Runtime：负责把工作区变成可检索证据，并把有限、带定位信息的结果交回 Agent。

因此这里说的“Agent pattern”，主要是“工具如何引导和约束 Agent 的下一步”，不是 zvec-grep 自己实现了一个 Agent。

## 1. 已确认的 Pattern

### Retrieval Routing as Agent Guidance

通过 MCP description、server instructions 和安装后的 Agent guidance，影响 Agent 选择检索路线。

### Hybrid Discovery Before Exact Verification

先用语义和词法混合检索发现相关区域，再用精确搜索验证具体符号或文本。

### Structured Evidence Instead of Raw Files

索引保留符号、section、路径和范围，结果返回有限证据，减少 Agent 上下文噪声。

### Eventual Search with Explicit Freshness

允许 Agent 使用当前索引，同时显式标记可能过期，并把后台刷新状态带回结果。

### Authorization as a Retrieval Branch

远程 Embedding 授权不是普通 MCP tool approval，而是会改变搜索路径的运行时分支。

### Evidence Package Instead of Hit List

搜索结果不是只返回文件名和分数，而是给出：

- `freshness` 和可能的 `background_refresh`；
- `matchedBy`，区分 FTS、vector 或两者；
- 文件相对路径、行范围、有限 source snippet；
- 代码/Markdown 的 symbol、section、scope 等结构元数据；
- 多 query 时的 group、coverage/global fill 选择原因；
- 必要时的 trace 和 diagnostics。

MCP 默认还会压掉完整 structuredContent，只返回面向 Agent 的紧凑文本和 bounded preview，避免每个命中重复携带完整 outline/source。

代码证据：`src/mcp/tools.ts:476-503`、`src/cli/format/context.ts:134-220`、`src/engine/service/types.ts:155-243`。

### Minimal Default Surface

默认 `agent` toolset 只注册 `zvec_grep_search`。`rg`、index、drop、index status、server status 只在 `full` toolset 中提供；搜索缺索引时，返回信息明确建议使用 exact-search fallback，而创建或重建持久索引要求显式授权。

这是一种权限和认知负担都收窄的设计：Agent 普通情况下只做证据检索，不负责自行维护长期索引。

代码/文档证据：`src/mcp/toolset.ts:1-20`、`src/mcp/tools.ts:334-625`、`docs/03-mcp.md`。

### One Probe for Semantic Existence

当用户只是问“本地是否存在概念相关内容”，又没有足够精确的锚点时，安装到 Agent 的 guidance 要求最多做一次 focused semantic probe；结果不相关就停止。这个规则把“发现证据”与“无限扩大搜索”分开，避免 Agent 因为语义命中不确定而反复调用工具。

代码/文档证据：`src/cli/install.ts:2227-2276`、`docs/01-agents.md`。

## 2. Agent 的典型决策树

```text
问题是否需要当前工作区证据？
  ├─ 否 → 不调用 zvec-grep
  └─ 是
      ├─ 已知精确词 / 路径 / regex？
      │    ├─ 是 → native Grep/rg（full MCP 时可用 zvec_grep_rg）
      │    └─ 否 → zvec_grep_search
      │
      ├─ 返回足够证据？
      │    ├─ 是 → 直接基于 snippet 和定位继续推理
      │    └─ 否 → 用结果里的路径/范围做 focused exact follow-up
      │
      ├─ freshness = possibly_stale？
      │    ├─ 探索任务 → 可继续使用，并记录状态
      │    └─ 必须反映最新文件 → wait_for_fresh
      │
      └─ 远程 embedding 需要授权？
           ├─ 允许 → 执行原路线
           ├─ local FTS → 改为 lexical-only，并关闭自动更新
           └─ cancel/无交互 → 停止，不静默授权
```

这里的“直接基于 snippet 继续推理”不是臆测：工具描述明确要求把 sufficient snippets 当作 already-read evidence；只有证据不足时才需要进一步读文件。

## 3. 失败、取消与重试的 Agent 语义

### 3.1 缺少索引不是“自动修复”

索引不存在时，daemon 返回 `INDEX_MISSING`，并明确提示使用可用的 exact-search fallback。默认 Agent toolset 不拥有创建索引的工具；即使是 full toolset，创建、重建和删除持久索引也要求用户明确要求。

这避免了 Agent 为了回答一次问题，偷偷下载模型、扫描目录并改变工作区持久状态。

代码证据：`src/daemon/backend.ts:442-451`、`src/daemon/runtime-manager.ts:87-91`、`src/mcp/tools.ts:334-388`。

### 3.2 Embedding 失败不会盲目重试

索引 Embedding 失败时，代码会先按错误类型分类：

- 限流、5xx、408、部分临时网络失败：可重试，并带退避和 jitter；
- API key、endpoint、401/403/404、模型或维度不支持：快速失败；
- 取消信号：终止当前任务，不当成普通业务错误重试；
- 批量失败时还会尝试 one-by-one，以定位具体 fragment，但最终仍会保留失败文件和失败原因。

最近的修复把失败文件、失败原因和后续建议 surfaced 出来，并对凭据、Authorization 等敏感字段做脱敏。对 Agent 来说，这比“工具超时后重复调用”更有用：它能区分临时服务问题、配置问题和用户取消。

代码证据：`src/engine/pipeline/indexing/index.ts:1262-1413`、`:1616-1682`、`src/engine/errors.ts:50-72`。

### 3.3 授权拒绝和宿主不支持交互要区分

如果用户实际拒绝了授权，系统应停止并说明没有发送远程数据；如果 MCP 宿主不支持 `elicitation/create`，这是交互能力缺失，不应被误判成 MCP server 断开、API key 缺失或普通 51500 错误。安装到 Agent 的 guidance 为 Qoder 等宿主提供了内置问答工具的兼容路径，但无头环境没有用户选择时必须停止。

代码/文档证据：`src/mcp/tools.ts:739-783`、`src/authorization/prompt.ts:14-49`、`src/cli/install.ts:2239-2242`。

## 4. 设计代价和限制

已确认或值得继续验证：

- 语义召回不相关时，Agent 是否容易重复搜索
- ranked sample 会不会遗漏 exhaustive evidence
- 后台刷新带来的“读操作副作用”是否会让工具语义变复杂
- 结构化抽取是否会偏向代码而弱化普通文本
- Agent guidance 与模型实际工具选择之间是否存在偏差
- `zvec_grep_search` 的 MCP annotation 是 `readOnlyHint: false`，因为搜索可能触发后台 reconciliation；但对 Agent guidance 来说，它仍然是默认的证据读取入口。这里存在“逻辑读操作 / 运行时副作用”的语义张力。
- indexed search 需要已有 index；没有 index 时不能让 Agent 默默创建持久化状态，只能切到精确搜索，或等待用户明确要求建索引。
- `zvec_grep_rg` 是 exhaustive 路线，但默认 Agent toolset 不暴露它；所以 Agent 是否能在需要穷举时自然切换，取决于宿主是否提供 native rg/full toolset。
- MCP 默认返回紧凑文本，结构化结果在某些配置下才提供；这减少上下文，但也意味着调用方不能默认依赖完整 JSON 字段。

### 4.1 产品适用性判断

目前更准确的定位是“Agent 的本地 Workspace Semantic Retrieval 插件”，而不是通用代码搜索或完整文档 RAG 系统。

它有价值的场景是：工作区较大且相对稳定，Agent 面对架构、关系、跨文件或未知位置的问题，无法先写出可靠的精确关键词。它可以先做一次语义发现，再把路径和范围交给精确工具验证。

它的鸡肋边界也很清楚：

- 代码中的精确查找、引用查找和穷举分析，`rg`、LSP、IDE 索引更可靠；
- PDF、Word 等二进制文档没有完整解析链路；
- 频繁保存会带来文件级重新抽取和 Embedding 成本；
- ranked sample 不是 exhaustive evidence；
- watcher 只是最终一致，不看未保存内容；
- 远程 Embedding 还增加授权、隐私和费用问题。

因此，如果目标是“帮开发者搜索代码”，它的必要性较弱；如果目标是“给 Agent 增加工作区级语义发现能力”，它是一个价值明确但范围较窄的基础设施组件。

## 5. 与 Hermes / Pi 的交叉比较

这里只比较 zvec-grep 作为外部 Tool / Retrieval Runtime 的部分，不把它误写成完整 Harness：

- 它如何注入 Context
- 它如何影响 Tool Loop 的下一步
- 它是否拥有独立状态
- 它如何处理失败、等待、授权和新鲜度

当前可先得到的对照结论：

- Hermes / Pi 这类 Harness 决定“何时调用工具、如何把结果放回模型上下文、下一轮是否继续”；
- zvec-grep 决定“工作区证据如何被抽取、索引、检索、裁剪，并附带哪些可信度/新鲜度/授权状态”；
- 因此二者的接口边界应该记成 `Agent Loop → MCP Search → Context Evidence → Agent Loop`，而不是把 zvec-grep 归类成另一个 Harness。

后续如果对照 Hermes / Pi，重点看四个接口问题：

1. Harness 是否把 `possibly_stale` 当普通文本，还是能消费它的结构化状态；
2. Harness 收到 input-required 授权请求时，是否能正确暂停并回传用户选择；
3. Harness 是否在“证据不足”后能做一次有边界的 follow-up，而不是泛化成循环；
4. Harness 是否保留路径、范围和 matchedBy 等 provenance，避免最终回答丢失证据来源。
