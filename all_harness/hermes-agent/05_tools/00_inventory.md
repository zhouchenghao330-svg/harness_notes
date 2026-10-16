# Hermes Core Tools 清单

> 状态：进行中。Tool System 框架、`read_file`、`write_file`、`patch` 和 `search_files` 已经共同阅读；其余核心工具尚待逐组共同阅读。
>
> 本清单不是 Hermes 所有工具的产品功能列表。它只选择与 Pi Core Tools 可直接比较、最能体现 Harness 文件与执行原语设计的工具。

## 1. 本轮核心工具

| Pi 对照          | Hermes 工具                  | Toolset               | 模型侧主要能力                   | 主要副作用                   | 当前状态  |
| -------------- | -------------------------- | --------------------- | ------------------------- | ----------------------- | ----- |
| `read`         | `read_file`                | `file`                | 读取文件并把内容转换成模型 Observation | 只读文件系统                  | 已共同阅读 |
| `write`        | `write_file`               | `file`                | 创建文件或写入完整内容               | 创建、覆盖文件                 | 已共同阅读 |
| `edit`         | `patch`                    | `file`                | 对已有文件应用结构化 Patch          | 修改文件                    | 已共同阅读 |
| `grep/find/ls` | `search_files`             | `file`                | 搜索文件名和文件内容                | 通常只读，但会扫描文件系统           | 已共同阅读 |
| `bash`         | `terminal`                 | `terminal`            | 执行 Shell 命令，获得输出和进程状态     | 任意本机或远程环境副作用            | 待共同阅读 |
| Bash 后台进程控制    | `process`                  | `terminal`            | 查询、轮询、输入或终止已有命令进程         | 改变运行中进程状态               | 待共同阅读 |
| 无直接对应          | `web_search`、`web_extract` | `web`                 | 搜索外部信息并提取网页内容             | 网络访问；把不可信外部内容带入 Context | 待共同阅读 |
| 无直接对应          | Browser Tools（13 个）        | `browser` 及相关 Toolset | 操作有状态浏览器并观察页面变化           | 网络访问、页面交互、登录态与外部操作      | 待共同阅读 |
| 无直接对应          | `execute_code`             | `code_execution`      | 在一次调用中用 Python 编排受限工具     | 批量执行多个工具，副作用由被调工具决定     | 待共同阅读 |
| 无直接对应          | `clarify`                  | `clarify`             | 向用户提问并等待回答后继续             | 暂停执行并引入人工输入             | 待共同阅读 |

主要注册入口：

~~~text
tools/file_tools.py
  read_file / write_file / patch / search_files

tools/terminal_tool.py
  terminal

tools/process_registry.py
  process
~~~

## 2. 为什么只选择这些工具

它们构成一个 Coding Agent 最基础的闭环：

~~~text
发现目标
  search_files
       ↓
读取证据
  read_file
       ↓
修改文件
  write_file / patch
       ↓
运行命令验证
  terminal
       ↓
长进程继续交互
  process
~~~

前五组与 Pi 的核心工具几乎一一对应；另外四组用于补足 Hermes 自己最有代表性的 Runtime 能力：

~~~text
web_search + web_extract
  外部不可信信息如何进入 Context

browser tools
  Stateful Tool 与交互环境

execute_code
  Meta Tool 与程序化工具编排

clarify
  Human-in-the-loop、等待和恢复
~~~

因此这套核心工具适合比较：

- 模型可见 Schema 如何设计；
- 文件读写和 Patch 的前置条件；
- 输出如何截断、外溢和重新取回；
- 文件操作是否有路径限制、冲突控制和写后校验；
- Shell 长任务如何从一次调用变成可继续管理的进程；
- 取消、超时、审批和副作用分别意味着什么；
- Tool Result 中哪些内容进入模型，哪些只进入 UI、日志或状态层。

## 3. 本轮明确不展开的工具

以下能力仍属于 Hermes，但不纳入本轮 Core Tools 逐项拆解：

- Memory、Session Search、Skills、Todo；
- Delegation、Kanban、Cron；
- 多模态生成与 TTS；
- Discord、Feishu、Yuanbao、Home Assistant 等平台工具；
- Desktop UI 与 Project 工具；
- Plugin、MCP 和 Context Engine 动态工具。

原因不是它们不重要，而是它们属于更高层业务能力、平台能力或已经在其他章节分析过的机制。当前目标是先读透最底层的文件和命令执行原语。

## 4. 拆解结构

~~~text
05_tools/
├── 00_inventory.md
├── 01_read_file.md
├── 02_write_file.md
├── 03_patch.md
├── 04_terminal_and_process.md
├── 05_search_files.md
├── 06_web_search_and_extract.md
├── 07_browser_tools.md
├── 08_execute_code.md
└── 09_clarify.md
~~~

其中：

- `terminal` 与 `process` 放在同一组，因为 `process` 不是独立命令执行器，而是长命令、后台命令和交互进程的后续控制面；
- Browser 的 13 个入口放在同一组，因为它们共享浏览器 Session、页面状态和元素引用生命周期；
- `execute_code` 单独研究，因为它是能在内部再次调用受限工具的 Meta Tool；
- `clarify` 单独研究，因为它改变的是 Agent 的等待、人工输入和恢复控制流。

## 5. 每个核心工具统一检查的问题

1. 模型看到的 name、description、parameters 和使用指导是什么？
2. 工具什么时候进入当前 Agent 的 Tool Snapshot？
3. 参数如何解析、规范化和验证？
4. 真正的文件、Shell 或进程后端在哪里？
5. 返回给模型的 content 与 UI / 日志 details 如何区分？
6. 大结果怎样截断、外溢和再次取回？
7. 有哪些文件、进程或网络副作用？
8. Approval、Guardrail、路径安全和危险命令检查在哪里？
9. 并发、资源冲突、取消、超时和失败怎样处理？
10. Plugin / Middleware 是否能覆盖、拦截或包装这次执行？
11. 与 Pi 同类工具相比，设计收益和代价是什么？

## 6. 当前结论边界

目前已经完成：

~~~text
核心工具范围确定
        +
Pi 对照关系确定
        +
共同阅读顺序确定
        +
read_file 的 Runtime 语义拆解
        +
write_file 的受控提交语义拆解
        +
patch 的条件修改与模糊匹配语义拆解
        +
search_files 的组合搜索与候选发现语义拆解
~~~

`read_file`、`write_file`、`patch` 和 `search_files` 已完成阶段性拆解；其中 `search_files` 的重点是用 `target` 在文件名搜索与内容搜索之间切换，并把结果塑形成后续 `read_file` 可继续精读的候选集合。其余具体工具仍待共同阅读。
