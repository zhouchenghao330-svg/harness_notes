# zvec-grep 04：搜索 Pipeline

> 状态：Query Group、召回和结果选择已完成第一轮确认。

## 1. 查询先被转换为中间请求

MCP handler 调用 `normalizeSearchInput()`，它不执行搜索，只把外部工具输入统一为 `NormalizedSearchInput`：

```text
MCP input
  ├─ query / queries
  ├─ fts / vector
  ├─ fuse
  └─ filters / freshness / autoUpdate
       ↓
NormalizedSearchInput
  ├─ primary query list
  └─ ordered routes: { mode: fts | vector, query }
```

`query` 和 `queries` 会形成 primary queries；`fts` 和 `vector` 会形成 supplemental routes。若 CLI 直接传入 `routes`，则保留显式 route 顺序。

代码证据：`src/mcp/input-normalization.ts:40-157`。

## 2. Query Group

`ZvecGrepService` 会把 primary query 和 supplemental route 组织成 Query Group：

```text
query: "authentication flow"
  → Q1 [primary]
      ├─ FTS: authentication flow
      └─ Vector: authentication flow

fts: ["AuthService"]
  → Q2 [supplemental]
      └─ FTS: AuthService
```

因此，一个自然语言 `query` 默认已经是 FTS + Vector 的混合检索，而不是只走向量搜索。

`fuse` 有两种结果形态：

- `fuse = false`：每个 Query Group 单独执行，保留 `groupResults`；
- `fuse = true`：在 Context 层把所有 groups 合成一个 Search Plan。

代码证据：`src/engine/service/zvec-grep.ts:1689-1800`、`src/engine/service/zvec-grep.ts:930-1009`。

## 3. 每个 Search Plan 内部的召回

`searchWorkspaceIndex()` 对一个 Search Plan 执行以下步骤：

1. 校验并规范化 routes；
2. 根据 path、file type、symbol 和修改时间生成过滤条件；
3. 对 Vector routes 调用 query Embedding；
4. 分别从 FTS 和 Vector 存储中召回候选；
5. 把同一个 entity 在不同 route 中的证据合并；
6. 用 RRF 计算融合分数并排序；
7. 将前 `limit` 个候选转换为 Search Hit。

代码证据：`src/engine/pipeline/search/index.ts:89-180`。

## 4. Adaptive Recall

召回深度不是固定等于最终返回数量：

- 初始深度为 200；
- 如果候选不够且底层结果仍然饱和，就扩大到 400、800，最多 2000；
- 目标候选数至少是 `limit * 5`，且不低于 50。

这样做是为了给后续融合留下足够候选，而不是只融合每条 route 的最终可见结果。

`preferSymbol` 开启时，项目还会从 query 中提取可能的 symbol name，追加一个带 symbol filter 的 FTS route。

代码证据：`src/engine/pipeline/search/index.ts:520-644`。

## 5. 第一层 RRF：FTS 和 Vector 融合

候选以 public entity id 去重。每条 route 给候选贡献：

```text
1 / (60 + route_rank)
```

同一个 entity 如果同时被 FTS 和 Vector 召回，就同时获得两条证据，最终 `matchedBy` 为 `fts+vector`。

代码证据：`src/engine/pipeline/search/index.ts:1147-1187`。

这里融合的是 rank，不是原始分数。FTS 的分数和向量相似度不会被校准到同一尺度，也没有 cross-encoder 或 LLM reranker。RRF 的效果主要是奖励“在多条路线中都靠前出现”的实体。

例如：

```text
A: FTS rank 1 + Vector rank 5
B: FTS rank 2（只被 FTS 找到）

score(A) = 1 / 61 + 1 / 65
score(B) = 1 / 62
```

因此 A 会因为得到两条独立路线的支持而占优。

## 6. 第二层：跨 Query Group 的结果选择

当 `fuse = false` 时，多个 group 的结果还会在 Context 层继续处理：

1. 按 entity 或文件范围去重；
2. 优先保证每个 primary group 都有代表结果；
3. 最多优先挑选 6 个 coverage items；
4. 再用全局排序补齐；
5. 给最终 Context Item 重新编号。

跨 group 的全局分数同样使用基于 rank 的 RRF 思路。

代码证据：`src/engine/service/zvec-grep.ts:1847-1915`。

具体的跨 group 分数是：

```text
GroupRRF(entity) = Σ 1 / (60 + entity 在各 group 中的 rank)
```

但最终选择不完全等价于全局分数排序：它先尽量保证 primary groups 的 coverage，最多优先选择 6 个 coverage items，再用 global fill 补齐，最后重新编号。

代码证据：`src/engine/service/zvec-grep.ts:1962-1979`。

## 7. Search Hit 到 Agent Context Item

对 Agent 可见的结果不是原始 zvec document，而是 `ZvecGrepContextItem`，包括：

- 文件绝对路径和相对路径
- 源码或文档范围
- 可展示的 excerpt range
- `content` 和 `contentRole`
- `outline`
- `rank`、`score`、`matchedBy`
- `entityId`、metadata 和 query group 信息
- 文件是否可能过期

这一步把底层检索结果转换成 Agent 可消费的 Evidence Context。

类型证据：`src/engine/service/types.ts:155-243`。

## 8. Agent 视角的设计含义

zvec-grep 实际上不是“给 Agent 一个搜索框”，而是：

```text
用户意图
  → 多条检索路线
  → 多层候选融合
  → 结构化去重和覆盖选择
  → 有界 Evidence Context
```

它的目标是让 Agent 用更少的文件读取和更少的上下文，先拿到足以判断相关性的证据。

## 9. 完整 retrieval 链路与评价

```text
query / queries
  → Q1、Q2 ... query groups
  → 每个 group 的 FTS route + Vector route
  → 初始 top-200 自适应扩大到最多 2000
  → 按 entity id 合并证据
  → 第一层 RRF 融合 FTS / Vector
  → 多 group 按 entity 去重
  → 第二层 Group RRF + primary coverage
  → 生成带路径、范围、snippet、metadata 的 Context Item
```

因此它是一套工程上简单、可解释的 hybrid retrieval：FTS 负责精确词法，Vector 负责语义相似，RRF 负责合并，Context 层负责去重和控制 Agent 上下文。

它不是特别智能的重排系统。它的价值主要在“没有明确关键词时先发现相关区域”，而不是替代 exhaustive `rg`、LSP 或完整影响分析。

## 10. 待继续确认

- fragment 为什么要恢复成 entity；
- 代码 symbol metadata 如何参与索引和 query；
- Markdown section 的 outline 如何处理；
- 最终 formatter 如何进一步截断源码片段；
- ranked sample 和 exhaustive `rg` 在 Agent 控制流中如何分工。
