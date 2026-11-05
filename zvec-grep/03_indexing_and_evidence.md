# zvec-grep 03：索引与证据单元

> 状态：第一轮已确认索引如何为 Agent 生成可检索证据。

## 1. 为什么索引属于 Agent 研究范围

索引不是单纯的数据工程。它决定 Agent 能发现什么、结果保留哪些结构，以及返回的证据是否足够判断相关性。

本项目的基本单位不是“文件”，而是 `EntityFragment`：带有文件 ID、范围、内容和可选 metadata 的局部证据单元。

## 2. 索引流程

```text
扫描文件
  ↓
新增 / 修改 / 删除 diff
  ↓
读取文件
  ↓
按文件类型抽取 fragments
  ↓
为 fragment 准备 Embedding 内容
  ↓
Embedding
  ↓
写入 zvec collection 和文件 metadata
```

主流程位于 `src/engine/pipeline/indexing/index.ts:427-529`；单文件准备、Embedding 和提交位于 `:899-1109`。

索引更新按 content hash、文件大小和修改时间计算 diff。删除的文件会从存储中清除，新增和修改的文件会重新生成 fragments。

### 2.1 文件不是无条件进入索引

扫描器会先应用 workspace root 的 `include` / `exclude`、glob、文件类型、深度和大小规则，再考虑文件类型 extractor。默认会跳过 `.git`、`.zvec-grep`、依赖/构建/缓存/生成目录，以及 lock、source map、minified、generated 等文件。

同时会遵循 `.gitignore` 和额外配置的 ignore 文件；隐藏路径默认不进入索引，符号链接默认不跟随。空文件、超过类型大小限制的文件、常见二进制文件也会被跳过，并计入 scan diagnostics。

默认大小限制按类型区分：code 约 1 MB、data 约 16 MB、image 约 10 MB、普通 text 约 256 MB。图片虽有 extractor，但默认的 raster 文件规则会把它们排除出语义索引；精确 rg 路径不受这个语义索引规则影响。

代码证据：`src/engine/pipeline/indexing/scanner/index.ts:49-120`、`:460-491`、`:1074-1153`、`src/engine/file-size-policy.ts:1-25`。

### 2.2 增量更新的粒度是文件

watcher 知道具体变更路径时，会把变化交给 path-level index pipeline；重新处理的是发生变化的文件或目录范围，而不是整个 workspace。文件内部并不会只重算一个字符：文件重新进入 extractor 后，原有 fragments 会被替换。

如果多个保存事件在 debounce 窗口内发生，ChangeSet 会合并同一个路径；索引任务开始时读取磁盘上的最终内容。因此它更像“对当前磁盘版本做增量重建”，而不是记录每个中间编辑版本。

代码证据：`src/daemon/watch-manager.ts:159-227`、`src/daemon/index-coordinator.ts:30-73`、`src/engine/pipeline/indexing/index.ts:663-714`。

## 3. Extractor 是证据建模层

`src/engine/extraction/runtime.ts` 根据 source 路由到四种 extractor：

- code
- markdown
- text
- image

JSON、YAML、CSV、TOML 等虽然被识别为 `data`，但当前没有独立的数据结构 extractor，最终主要走普通文本切块。未知但看起来像文本的扩展名也会按 text 处理。

`extractForIndexing()` 对代码额外返回 `embeddingSource`，允许“给 Agent 展示的内容”和“用于向量化的内容”有差异。

代码证据：`src/engine/extraction/runtime.ts:15-67`。

## 4. 代码证据单元

### 4.1 先识别代码实体

`CodeExtractor` 使用 Tree-sitter parser 和语言 adapter 遍历 AST，识别：

- module
- class
- interface
- function
- value
- alias

每个实体同时收集：

- symbol name
- scope breadcrumb
- signature
- doc
- modifiers
- 原始 source range

代码证据：`src/engine/extraction/code/extractor.ts:230-279`。

### 4.2 小实体和大实体不同处理

如果实体不大，就生成一个包含完整源码的 fragment。

如果实体过大，就生成：

```text
major entity fragment
  ├─ outline：header、members 或 calls
  └─ source window fragments：按语句或行切分，带 overlap
```

这些 fragment 通过 `group` 关联。搜索命中某个 source window 后，检索层可以恢复到 major entity；Agent 最终同时得到结构轮廓和命中的源码证据。

代码证据：`src/engine/extraction/code/extractor.ts:281-338`、`:676-713`。

### 4.3 代码 metadata 也参与向量化

Embedding 内容会在源码前加入压缩后的 metadata，例如：

```text
symbol: function loadConfig
scope: App::Config
signature: ...
modifiers: exported async
doc: ...
```

metadata 预算最多占 chunk 的 25%，避免结构信息完全挤掉源码内容。

代码证据：`src/engine/extraction/vector-content.ts:6-27`、`:68-109`。

## 5. Markdown 和普通文本

### 5.1 Markdown

Markdown 先扫描 heading，再构造 section 和 breadcrumb：

```text
## Runtime
  ↓
### Freshness
  ↓
scope: Runtime::Freshness
```

每个 section 有 heading、level 和 scope metadata。过长 section 会生成一个 outline fragment，再生成多个带 group 的内容窗口。

代码证据：`src/engine/extraction/markdown/extractor.ts:34-126`、`:247-286`、`:553-559`。

### 5.2 普通文本

普通文本使用按行的 chunk 和 overlap，并保留准确的 text range；它没有代码 symbol 或 Markdown heading metadata。

代码证据：`src/engine/extraction/text/extractor.ts:65-179`。

## 6. 写入存储

每个 fragment 最终写成 zvec document，保存：

- `embedding` 向量字段
- `text` 字段，用于 FTS
- fragment 的 range、group、metadata
- file ID 和文件 metadata

每个文件替换索引时，旧 documents 先删除，再批量 upsert 新 documents。

代码证据：`src/engine/storage/zvec.ts:206-260`。

## 7. 这里的“模型”不是 Agent 的推理模型

zvec-grep 使用的是 Embedding Model，不负责生成回答。它只把 document fragment 或 query 转成向量：

- 建索引时调用 `model.embed(..., { purpose: "document" })`；
- 查询向量路线时调用 `model.embed(..., { purpose: "query" })`；
- 同一个模型的 query/document purpose 可能使用不同前缀或指令格式，例如本地 Qwen embedding 会为 query 加检索指令；
- 只有 vector route 需要 Embedding model，FTS route 不需要调用模型。

代码证据：`src/engine/pipeline/indexing/index.ts:1367-1375`、`src/engine/pipeline/search/index.ts:432-454`、`src/engine/models/backends/llama-cpp.ts:689-703`。

因此完整的 Agent 分工是：

```text
Agent LLM：理解用户问题、决定是否检索、消费证据、生成回答
        ↑
zvec-grep：把 query/document 变成可比较的向量，并返回定位后的证据
```

模型运行时还会被 daemon 的 `EmbeddingModelPool` 复用：连续搜索可以命中已加载模型，空闲一段时间后再回收。这个缓存优化的是检索运行成本，不改变 Agent 的 session 或 memory。

代码证据：`src/daemon/model-pool.ts:45-124`、`:156-219`。

## 8. 索引状态对 Agent 的影响

- 文件变化会进入 added / modified / deleted diff；
- Embedding 或抽取失败会记录为 failed file；
- 输入过长可能产生 truncated fragment 统计；
- watcher 发现变化后，搜索结果可能标记为 `possibly_stale`；
- 没有索引时，Agent 应该转向精确 Grep / rg，而不是自行静默创建索引。

## 9. Agent 视角的核心结论

zvec-grep 的索引实际上在做一层“证据建模”：

```text
原始文件
  → 结构实体
  → 可向量化的语义表示
  → 可定位的 source fragment
  → 可恢复的 Agent Context
```

因此，索引质量不只是影响召回率，也影响 Agent 能否理解命中内容属于哪个函数、类、section 或 scope。
