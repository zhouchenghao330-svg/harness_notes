# zvec-grep 05：Runtime、新鲜度与授权

## 1. 先给结论

这一层解决的不是“怎么搜”，而是 Agent 搜索时的两个运行时问题：

1. 当前索引是否足够新，能不能先返回结果；
2. 如果要用远程 Embedding，哪些数据可以离开本机，以及用户是否明确授权。

所以 `zvec-grep` 的搜索工具实际上是一个“检索 + 工作区 Runtime 协调器”，不是一次简单的数据库查询。

## 2. Server / Direct / Auto

从 Agent 视角，核心差异是 Runtime 是否复用：

- Server 模式由 daemon 持有工作区 Runtime，可以复用 watcher、模型池和读会话；
- RootRuntime 在搜索时优先使用 writer context，否则从 model pool 借模型并打开或复用 read session；
- 搜索操作会被串行化，避免同一个工作区的索引读写互相踩踏；
- MCP HTTP 入口默认使用 `agent` toolset，daemon 根据工作区激活对应 RootRuntime。

代码证据：`src/daemon/backend.ts:432-589`、`src/daemon/root-runtime.ts:97-151`、`src/daemon/root-runtime.ts:347-375`。

这意味着 Agent 连续发起多次检索时，不必每次都重新初始化模型和索引；工作区的变化由 watcher 维护，而不是由每次搜索临时猜测。

## 3. RootRuntime

已确认的结构：

- watcher 收集文件变化，并把变化合并到 `ChangeSet`；
- `IndexCoordinator` 将窄范围变化转成增量索引任务，无法安全缩小范围时转成全量 reconciliation；
- Runtime 用 dirty/indexed revision 追踪“文件系统看到的版本”和“索引已经处理到的版本”；
- writer context 用于索引写入，reader context 用于搜索，二者通过 Runtime 生命周期协调；
- `.gitignore` 变化会触发目录级重扫，因为它可能改变可索引文件集合。

代码证据：`src/daemon/index-coordinator.ts:22-75`、`src/daemon/change-set.ts:18-170`。

## 4. Freshness

### 4.1 默认策略：先搜，后台追平

公开搜索参数默认是 `freshness: "eventual"`、`autoUpdate: true`。因此搜索可以直接使用当前可读索引；如果 Runtime 发现 watcher 或变更记录表明索引可能落后，同时允许自动更新，就提交一个不阻塞当前请求的 `background_reconcile` 任务。

搜索响应会带上 freshness 状态和 indexing snapshot。只要存在已知变更、watcher 尚未处理完，或相关索引任务仍在运行，结果就可能标成 `possibly_stale`；否则标成 `fresh`。

代码证据：`src/daemon/backend.ts:432-589`、`src/daemon/backend.ts:876-925`。

### 4.2 强一致选项：`wait_for_fresh`

当调用方明确要求 `wait_for_fresh` 时，Backend 会等待已知变化稳定下来；如果仍需要索引，就提交 `fresh_query` 任务并等待它完成，失败或取消会向上抛出，而不是悄悄返回旧结果。

代码证据：`src/daemon/backend.ts:1060-1112`。

### 4.3 对 Agent 的含义

Agent 不需要先调用 `status` 再搜索。一次搜索响应已经带有“结果 + 新鲜度证据”，Agent 可以根据任务重要性决定：

- 探索、定位、理解架构：接受 `possibly_stale` 的结果继续推理；
- 需要确认刚写入的内容：下一次搜索明确要求 `wait_for_fresh`；
- 发现结果与预期不符：再检查 freshness/indexing snapshot，而不是立即把搜索失败归因于召回能力。

这是一个典型的“检索结果和运行时状态一起返回”的 Agent 工具设计。

### 4.4 watcher 不是逐字符实时索引

Server 激活工作区 Runtime 后才建立 watcher。文件变化会先进入 `ChangeSet`，默认 debounce 约 750ms，连续变化最多等待约 5s；同一路径的多次变化会被合并，索引任务开始时读取磁盘上的当前版本。

因此：

- 未保存的编辑器内存内容不会被看见；
- 快速“改一个字、撤销、再保存”可能只索引最终版本；
- 如果中间保存后已经过了 debounce，可能先索引中间版本，之后再增量索引最终版本；
- 搜索发生在更新期间时，默认仍可读旧索引并返回 `possibly_stale`。

watcher 事件之外，Server 默认约每小时做一次完整 reconciliation，用来修复可能漏掉的文件系统事件；watcher 出错、恢复运行或事件数量过大时也可能升级为 full reconciliation。

代码证据：`src/daemon/watch-manager.ts:187-227`、`:254-293`、`src/daemon/backend.ts:927-1014`；文档证据：`docs/06-server.md` 的 Refresh behavior。

所以它提供的是“后台 eventually consistent workspace index”，不是 IDE 那种逐字符实时语义索引。

## 5. Remote Embedding Authorization

### 5.1 什么时候进入授权分支

本地模型或本地 FTS 不需要这条授权流程。只有选择 qwen 等远程 Embedding provider 时，Backend 才会先规划授权；如果工作区已经有匹配的 permit，则复用它，否则通过 MCP elicitation 向用户展示授权选项。

代码证据：`src/daemon/backend.ts:218-280`、`src/mcp/tools.ts:724-838`。

### 5.2 用户可选择的路线

授权表单的控制流不是简单的“同意 / 拒绝”，而是四路分支：

- `allow_once`：只允许这一次操作；
- `allow_workspace`：给当前工作区建立可复用授权；
- `use_local_search`：放弃远程向量检索，改走本地 FTS fallback（只有调用方提供了 alternative 时才出现）；
- `cancel`：取消本次操作。

本地 fallback 会把所有查询路线改成 FTS，并关闭自动更新、使用 `eventual` freshness；它因此是“改变检索路线”，不是把远程请求重试一遍。

代码证据：`src/mcp/tools.ts:840-853`。

### 5.3 授权保护和持久化

- 当前 permit 放在 AsyncLocalStorage 中，底层模型调用会校验 provider、model、endpoint 和 capability，防止授权被错误复用；
- workspace 级 grant 持久化在 `<workspace>/.zvec-grep/authorization.json`；
- grant 使用本机 `~/.zvec-grep/authorization-signing.key` 签名并验证，避免直接信任可编辑的授权文件；
- disclosure 会说明可能发送 query text 以及所选或发生变化的 workspace 文件，且可能产生费用。

代码证据：`src/authorization/operation.ts:15-77`、`src/authorization/manager.ts:9-29`、`src/authorization/store.ts:27-95`、`src/authorization/prompt.ts:16-49`。

这里要区分两种数据边界：搜索 query 的远程 embedding 需要发送查询文本；索引 document 的远程 embedding 可能发送被选中的或发生变化的 workspace 内容。具体 disclosure 以当前操作的授权提示为准，不能笼统说“整个目录都会上传”。

## 6. 对 Agent Runtime 的关键启示

```text
Agent 调用搜索
    ↓
判断 freshness：直接读 / 等待追平 / 后台 reconciliation
    ↓
判断 embedding 路线：本地 / 远程授权 / 本地 FTS fallback
    ↓
返回结果，同时返回 freshness、indexing snapshot、诊断信息
    ↓
Agent 决定是否继续推理、换路线或要求更强一致性
```

因此，`zvec-grep` 给 Agent 暴露的不只是“搜索 API”，还有两个重要的运行时事实：

1. 这个结果来自哪个新鲜度状态；
2. 这个结果是否经过了用户授权的数据边界。

这也是后面阅读 `06_agent_patterns_and_open_questions.md` 时最值得保留的主线。
