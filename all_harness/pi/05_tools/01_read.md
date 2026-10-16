# Pi Concrete Tool 01：read

> 状态：已共同阅读并确认。

## 1. 工具定位

`read` 的职责不是简单包装 `fs.readFile()`，而是把文件系统中的内容转换成模型可以消费的 Observation：

~~~text
文件内容
  ↓
路径解析与可读性检查
  ↓
文本 / 图片识别
  ↓
分页、截断或图片处理
  ↓
AgentToolResult.content
  ↓
下一次模型调用
~~~

它是 Pi 默认 Coding Tools 和 Read-only Tools 都包含的基础工具。

## 2. 模型可见接口

~~~typescript
read({
  path: string,
  offset?: number,
  limit?: number
})
~~~

- `path`：相对路径或绝对路径。
- `offset`：从第几行开始，使用 1-based 行号。
- `limit`：最多读取多少行。

工具 Schema 通过 `Context.tools` 进入 Provider 请求。System Prompt 另外加入：

~~~text
Read file contents
Use read to examine files instead of cat or sed.
~~~

因此 Schema 告诉模型参数如何填写，Prompt guideline 告诉模型什么时候优先使用 `read`。

## 3. 内部执行流程

~~~text
read(path, offset?, limit?)
        ↓
resolveReadPathAsync(path, cwd)
        ↓
operations.access(absolutePath)
        ↓
detectImageMimeType()
   ┌────┴────┐
   ↓         ↓
文本文件     图片
   ↓         ↓
UTF-8       processImage()
offset      自动缩放
limit       ImageContent
truncateHead
   └────┬────┘
        ↓
AgentToolResult
├── content：进入模型 Context
└── details.truncation：供 UI 和运行时使用
~~~

取消采用 `AbortSignal`。底层 `fs.readFile()` 没有被真正强制终止；Harness 在异步步骤之间检查取消状态，并停止接纳迟到结果。

## 4. 文本读取与 Context 控制

文本输出受两个独立上限控制：

~~~text
最多 2000 行
最多 50 KB
先达到哪个限制，就按哪个截断
~~~

`read` 使用 `truncateHead()` 保留头部完整行，不正常返回半行。截断后会在 `content` 中加入可执行的续读提示：

~~~text
[Showing lines 1-800 of 4000. Use offset=801 to continue.]
~~~

如果第一行本身就超过 50 KB，工具返回提示，让模型通过 `bash` 使用 `sed` 和 `head` 定点读取。

这里需要和 Bash 输出治理区分：

> `read` 截断后不会把完整内容复制到临时文件。原文件本身仍然存在，模型通过 `offset` 继续读取即可。

## 5. 图片读取

`read` 同时支持基础图片读取：

~~~text
jpg / png / gif / webp / bmp
        ↓
检测 MIME
        ↓
默认自动缩放到最大 2000×2000
        ↓
TextContent + ImageContent
~~~

如果当前模型不支持图片，结果中会加入说明；图片不会作为有效视觉输入交给该模型。

## 6. 路径解析

`cwd` 用于解析相对路径，但它不是权限沙箱。`read` 接受相对路径和绝对路径，并支持 `~`、`@` 前缀及 Unicode 空格规范化。

读取路径还会尝试兼容 macOS 常见文件名差异：

- 截图名称中 AM / PM 前的窄不换行空格；
- macOS 文件名的 NFD Unicode 形式；
- 直引号和弯引号差异。

只要 Pi 进程拥有操作系统读取权限，绝对路径可以指向 `cwd` 之外。原生 `read` 本身不提供工作区隔离。

## 7. ReadOperations：工具语义与执行后端分离

真正的读取动作被抽象为：

~~~typescript
interface ReadOperations {
  readFile(absolutePath): Promise<Buffer>
  access(absolutePath): Promise<void>
  detectImageMimeType?(absolutePath): Promise<string | null | undefined>
}
~~~

默认实现是本机文件系统：

~~~text
access   → fs.access(R_OK)
readFile → fs.readFile()
MIME     → 本地文件类型检测
~~~

`ReadOperations` 是创建工具时由 Harness、SDK 或 Extension 注入的后端依赖，不是模型参数。

### 7.1 path 不能动态选择后端

模型不能使用这些 Pi 原生格式：

~~~text
ssh://server/path/file
docker://container/path/file
remote:server:/path/file
~~~

`read` 的 Schema 没有 `host`、`containerId`、`backend`、`transport` 或 `connectionId`。模型填写的 `path` 只表示当前工具实例所绑定文件系统中的路径。

决定后端的是宿主创建工具时传入的配置：

~~~typescript
createReadTool(cwd, {
  operations: someReadOperations
})
~~~

因此准确关系是：

~~~text
模型调用 read(path)
        ↓
已经创建好的 read 工具实例
        ↓
该实例预先绑定的 ReadOperations
        ↓
本地文件系统或宿主指定的其他后端
~~~

### 7.2 Pi 默认行为与可选 SSH Extension

普通 Pi CLI 不加载相关 Extension 时：

~~~text
read(path)
  → defaultReadOperations
  → 本机文件系统
~~~

此时如果模型需要读取远程主机，通常只能使用 `bash` 执行显式 SSH 命令，例如：

~~~text
ssh user@host 'cat /path/file'
~~~

但是 Pi 仓库确实提供了可选示例：

~~~text
packages/coding-agent/examples/extensions/ssh.ts
~~~

启动方式：

~~~bash
pi -e ./ssh.ts --ssh user@host:/remote/path
~~~

这个 Extension 会注册同名工具覆盖默认的 `read/write/edit/bash`。当 `--ssh` 存在时，它创建绑定到该 SSH 主机的 Operations：

~~~text
模型：read("src/app.ts")
        ↓
本地路径解析
        ↓
Extension 把 localCwd 前缀映射为 remoteCwd
        ↓
ssh user@host 'cat /remote/path/src/app.ts'
~~~

因此这项能力应分成三种事实：

1. **默认行为**：内置 `read` 读取本机文件；
2. **架构能力**：宿主可以注入其他 `ReadOperations`；
3. **仓库示例**：官方 SSH 和 Gondolin Extension 已经演示如何重绑定整套工具后端。

它仍然不是由 `path` 动态选后端，而是由 Extension、Session 配置和工具创建逻辑提前决定当前调用使用哪个 Operations。

### 7.3 为什么不能写成“read 原生支持 SSH”

“支持可注入 SSH 后端”和“工具原生识别 SSH 地址”是两种不同能力：

~~~text
当前设计：
宿主绑定 SSH backend
模型继续调用 read(path)

并不存在：
模型调用 read(ssh://host/path)
工具按照 URI 动态路由
~~~

这一设计模式可以记录为：

> Tool semantic 与 Tool backend 分离，但 backend selection 不属于模型侧工具协议。

## 8. 并发与一致性边界

`read` 不进入 `fileMutationQueue`。Pi 默认 Tool Batch 允许未声明 `executionMode: sequential` 的工具并发，因此同一批中存在：

~~~text
write("a.ts")
read("a.ts")
~~~

`read` 理论上可能读到写入前或写入后的状态。Pi 没有在原生文件工具层实现 reader / writer 路径冲突规划，也不提供事务性文件快照。

这与 Hermes 按路径重叠切分并发段和顺序屏障的设计形成直接对比。

## 9. UI 与结果分层

模型主要消费 `AgentToolResult.content`。`details.truncation` 保存结构化截断信息，TUI 用它显示行数、字节限制和警告。

TUI 还会：

- 按文件扩展名做语法高亮；
- 折叠时隐藏或限制正文；
- 对 `SKILL.md`、Pi docs、`AGENTS.md` 等资源文件显示更紧凑的分类标签。

这些展示能力不会进入 Provider 工具实现。

## 10. 设计取舍

### 优点

- 模型接口只有三个参数，稳定且容易调用；
- 对大文件自动分页，保护 Context；
- 文本和图片共用一个观察入口；
- 路径处理针对模型和 macOS 文件名做了容错；
- Operations 允许宿主替换执行环境，而不用修改模型协议。

### 代价和限制

- 原生工具没有 cwd 级权限隔离；
- 不能通过 `path` 动态选择 SSH、容器或其他后端；
- 截断内容不产生完整结果副本，只能继续读取原文件；
- 不与 `write/edit` 建立 reader / writer 锁；
- 不是事务性快照；
- 工具取消不能保证底层文件系统操作已经停止。

## 11. 代码证据

~~~text
模型 Schema、Prompt 和执行：
packages/coding-agent/src/core/tools/read.ts

路径处理：
packages/coding-agent/src/core/tools/path-utils.ts

截断规则：
packages/coding-agent/src/core/tools/truncate.ts

工具集合与默认注册：
packages/coding-agent/src/core/tools/index.ts

可选 SSH 后端示例：
packages/coding-agent/examples/extensions/ssh.ts

可选 Gondolin VM 后端示例：
packages/coding-agent/examples/extensions/gondolin/index.ts

Extension 文档：
packages/coding-agent/docs/extensions.md

Tool Batch 调度：
packages/agent/src/agent-loop.ts
~~~

