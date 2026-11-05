# zvec-grep 02：Agent 接入与 MCP

> 状态：第一轮已确认
>
> 本章只记录 zvec-grep 如何进入 Agent、如何影响 Agent 的检索决策，以及 MCP 如何把检索请求交给运行时。

## 1. 接入不是单一配置，而是三层契约

```text
Agent 配置
  ├─ MCP Server 地址或 stdio 启动命令
  └─ Agent guidance / 权限配置

MCP Server
  ├─ server instructions
  ├─ tool descriptions
  └─ registered tools

Search Runtime
  └─ 接收 zvec_grep_search 并返回本地证据
```

zvec-grep 不负责调用大模型。它通过 Agent host 的配置、持久化指导文本和 MCP 元数据，影响上层 Agent 是否调用它以及选择哪条检索路线。

## 2. `zg install` 的入口和目标

代码入口：`src/cli/install.ts:136` 的 `runInstall()`。

当前内置 Agent Installer：

- Claude Code
- Codex
- OpenCode
- Cursor
- Qwen Code
- Qoder

每个 Installer 负责处理对应 Agent 的 MCP 配置、指导文件和权限格式。`runInstall()` 完成选定 Agent 的安装后，还会尝试启动 zvec-grep Server，并提示用户重启 Agent 或新建 Session。

## 3. 写入内容

### 3.1 MCP 配置

例如 Codex 会写入：

- `~/.codex/config.toml`
- `~/.codex/AGENTS.md`

Codex 的 MCP 配置支持两种连接方式：

- `stdio`：Agent 启动 `zg` 作为本地 MCP 进程；
- `http`：Agent 连接本地 Server URL。

配置块使用 `ZVEC_GREP_START` / `ZVEC_GREP_END` 标记，卸载时只移除 zvec-grep 管理的部分。

### 3.2 Agent guidance

`agentGuidanceBlock()` 生成一段 Agent 行为指导，并按不同宿主替换工具名。

核心路由规则：

```text
已知字符串、符号、路径、配置项、正则
  → native Grep / rg

位置未知，需要语义发现、关系、架构或跨文件综合
  → zvec_grep_search

既有精确锚点，又需要跨文件理解
  → 先 zvec_grep_search，再精确搜索验证
```

指导文本还要求：

- 每次调用传入 daemon 可见的绝对 `root`；
- 结果包含足够 snippet 时，不要重复读取整个文件；
- 读取 `freshness` 和 `background_refresh`，不要无条件先查 status；
- 不要静默创建、重建或删除持久化索引。

代码证据：`src/cli/install.ts:2227-2276`。

## 4. MCP toolset

默认 toolset 是 `agent`，只注册：

- `zvec_grep_search`

`full` toolset 额外注册：

- `zvec_grep_rg`
- `zvec_grep_index`
- `zvec_grep_index_drop`
- `zvec_grep_index_status`
- `zvec_grep_server_status`

默认只给 Agent 检索能力，不让 Agent 自行管理持久化索引。索引生命周期通常由用户通过 CLI 管理。

代码证据：`src/mcp/toolset.ts:1-20`、`src/mcp/tools.ts:334-625`。

## 5. `zvec_grep_search` 的 Agent 合同

工具至少需要：


```json
{
  "root": "/absolute/workspace/path",
  "query": "问题或检索意图"
}
```

支持的查询入口：

- `query`：一个 primary hybrid group；
- `queries`：多个 primary hybrid groups；
- `fts`：补充的 lexical route；
- `vector`：补充的 semantic route；
- `fuse`：是否把全部 query groups 合成一个 ranked plan。

默认值：

- `freshness = eventual`：优先立即使用当前索引；
- `autoUpdate = true`：允许搜索触发后台刷新。

代码证据：`src/mcp/schemas.ts:78-161`、`src/mcp/schemas.ts:241-260`。

## 6. Agent 侧的关键设计判断

### Retrieval Policy 是工具的一部分

工具 description、MCP instructions 和安装后的 Agent guidance 共同构成检索策略。zvec-grep 不仅提供“搜索能力”，还规定什么情况下应该搜索、什么时候停止搜索，以及何时转向精确验证。

### 默认工具面是有意收窄的

默认 Agent 只能调用搜索，不拥有索引写入和删除能力。这减少了 Agent 为了回答问题而擅自改变持久化状态的可能性。

### 搜索不是绝对纯读

`zvec_grep_search` 可以因为 `autoUpdate` 调度后台索引刷新，因此 MCP 注册时虽然标记为非 destructive，但不是严格意义上的无副作用读取。

### MCP approval 和 Remote Embedding authorization 分离

批准 Agent 调用本地 MCP Server，不等于允许把 query 或 workspace 内容发送给远程 Embedding provider。远程 Embedding 需要单独的授权分支。

## 7. 待继续确认

- 不同 Agent host 如何实际消费 server instructions 和 guidance；
- stdio bridge、HTTP endpoint 与 Agent Session 的连接生命周期；
- Remote Embedding authorization 如何在 MCP 请求中暂停和恢复；
- `zvec_grep_search` 的 compact text 如何进入上层 Agent 的下一轮 context。
