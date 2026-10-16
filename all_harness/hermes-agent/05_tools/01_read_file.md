# Hermes Core Tool 01：read_file

> 状态：已共同阅读。
>
> 核心定位：`read_file` 不是一个简单的文件读取函数，而是 **Filesystem → Model Context 的受控投影器，同时也是文件状态一致性协议里的 read side**。

## 1. 先给结论

如果只从功能表面看，`read_file` 做的是：

~~~text
path
  ↓
open / shell read
  ↓
string
~~~

但 Agent Runtime 真正需要的是：

~~~text
真实文件系统对象
        ↓
绑定到当前 Agent 所处的执行环境
        ↓
转换成有边界、可定位、适合模型消费的 Observation
        ↓
记录“模型已经看过什么”
        ↓
为后续 patch / write 建立版本证据
~~~

因此，Hermes 的 `read_file` 同时参与了多套协议：

~~~text
Environment Protocol
Context-budget Protocol
Epistemic-state Protocol
Concurrency Protocol
Mutation-consistency Protocol
Security Protocol
Cancellation Protocol
~~~

这也是本章最重要的认识：

> 成熟 Agent Harness 的价值，往往不在 `sed` 或 `open()` 本身，而在一次读取发生前后，Runtime 怎样管理这项能力的完整生命周期。

## 2. 四层 Runtime 模型

可以把 Hermes 的 `read_file` 拆成四层：

~~~text
① Environment Layer
   这个 path 属于哪个执行世界？
                ↓
② Observation Projection Layer
   文件怎样变成模型可消费的 Observation？
                ↓
③ Cognitive / Epistemic State Layer
   Runtime 怎样记录模型已经看过什么？
                ↓
④ Consistency / Scheduling Layer
   读取怎样约束后续修改，并避免并发竞态？
~~~

前两层解决“读到什么、返回什么”，后两层解决“模型知道什么、以后还能安全地做什么”。后两层才是 Hermes 这项工具最值得学习的部分。

---

## 3. 第一层：Environment Layer

这一层回答：

> `read_file("src/a.py")` 到底读取哪个机器、哪个容器、哪个工作目录里的文件？

### 3.1 文件工具必须和 Terminal 看到同一个世界

Hermes 没有让 `read_file` 永远使用 Python 主进程的 `open()`，而是把文件操作绑定到当前 Task 的 Terminal Environment：

~~~text
Agent Task / Session
   │
   ├── cwd
   ├── terminal session
   └── filesystem namespace
            │
            ├── Local
            ├── Docker
            ├── SSH
            ├── Singularity
            ├── Modal / Daytona
            └── Vercel Sandbox 等
~~~

真正的文件读取通过 `ShellFileOperations` 适配到当前 Terminal Environment。这样 terminal 和 file tools 使用同一个 cwd、同一个文件系统命名空间。

否则会出现严重的认知分裂：

~~~text
terminal:
  cd /tmp/worktree-123

read_file:
  却仍从宿主进程 /home/project 读取 foo.py
~~~

模型会以为自己正在操作 worktree，实际却读写了另一个仓库。

因此更准确的关系是：

> Terminal Environment 是底层“执行世界”，`read_file` 是这个世界上的结构化 Observation 接口。

### 3.2 相对路径如何解析

Hermes 对相对路径解析 cwd 时，大致按以下优先级选择：

~~~text
1. 当前 session 的 live terminal cwd
2. task / session 注册的 cwd override
3. 非占位值的 TERMINAL_CWD
4. Runtime 进程 cwd
~~~

`""`、`.`、`./`、`auto`、`cwd` 等占位值不会被误当成可靠的绝对工作目录。

容器和远程路径使用 POSIX 路径语义进行规范化，而不会先在宿主机上做 symlink 解析，否则宿主机可能根本不存在同一条路径。

### 3.3 这不是 Workspace Sandbox

Hermes 允许读取绝对路径，`read_file` 的路径解析并不等于“只能访问工作区”。

它依赖的是：

~~~text
当前 Environment 的可达范围
        +
敏感路径拦截
        +
结果脱敏
        +
外围 Middleware / Policy
~~~

所以 Environment Binding 和 Workspace Confinement 必须分开理解：前者保证“读对世界”，后者才决定“这个世界里允许读哪里”。

---

## 4. 第二层：Observation Projection Layer

这一层回答：

> 任意 Filesystem Object 怎样变成有边界、可定位、适合进入模型 Context 的 Observation？

模型面对的不只有普通代码文件，还可能是：

~~~text
238 KB Python
20 MB JSON
单行 5 MB 的压缩日志
UTF-16 文本
DOCX / XLSX / PDF
PNG
ELF / ZIP
设备文件 / FIFO
~~~

Runtime 不能简单地把所有原始字节塞进下一轮请求。因此 Hermes 实际建立的是：

~~~text
arbitrary filesystem object
            ↓
bounded textual observation
~~~

### 4.1 模型侧 Schema

模型看到的主要参数是：

~~~text
path    必填
offset  从第几行开始，1-based
limit   最多读取多少行
~~~

这套接口把读取单位统一成“行窗口”，便于模型继续分页，也方便后续 `patch` 引用位置。

这里存在一个当前实现上的契约漂移：

- Schema 声明的 `limit` 默认值是 `2000`；
- Handler 在调用参数缺失时实际使用 `500`；
- Dispatcher 不会自动把 JSON Schema default 注入参数。

所以模型省略 `limit` 时，当前真实行为是读取 500 行，而不是 Schema 暗示的 2000 行。这个问题不是读取算法问题，而是 **Tool Contract 和 Runtime Default 不一致**。

### 4.2 普通文本读取

普通文本路径大致是：

~~~text
normalize offset / limit
        ↓
解析当前 Environment 与绝对路径
        ↓
特殊文件与敏感路径检查
        ↓
后端按行读取
        ↓
单行截断
        ↓
加行号
        ↓
单次读取总字符预算
        ↓
敏感信息脱敏
        ↓
返回 JSON 字符串
~~~

底层通常利用当前 Terminal Environment 执行 Shell 范围读取，而不是先把整个文件搬到 Python 进程。这样可以尽量在信息源头控制传输量。

### 4.3 为什么使用紧凑行号

内容会被格式化为：

~~~text
120|def foo():
121|    ...
~~~

而不是用大量左侧空格对齐。

这不是纯粹的显示偏好。Agent Tool Result 的每个字符最终都可能进入下一次模型推理，因此格式设计要考虑 **Context Economics**：

~~~text
传统 API：返回越完整越好

Agent Tool：返回足以支持下一步决策的最小信息
~~~

行号仍然保留，因为它为定位、讨论和修改提供价值；纯装饰性空格没有认知价值，所以被省略。

### 4.4 分层预算

Hermes 不是只在最上层设置一个 `max_output_tokens`，而是在多个层级限制膨胀：

~~~text
单行长度预算
       ↓
单次 read_file 字符预算
       ↓
单轮所有 Tool Result 聚合预算
       ↓
整个 Conversation Context Budget
~~~

当前重要边界包括：

- `offset` 最小为 1；
- `limit` 受配置中的最大行数约束，默认上限为 2000；
- 单行默认最多保留约 2000 字符，超出后标注 `[truncated]`；
- 单次 `read_file` 默认总字符上限约 100,000，可配置；
- 大文件且读取范围过宽时，会给模型“缩小范围”的提示。

这条原则非常值得复用：

> 尽量在信息产生源附近完成第一轮预算控制，不要等到 Context 已经爆炸后才统一裁剪。

否则即使最终 Context 被截断，前面的远程传输、序列化、日志、回调和内存成本都已经发生。

### 4.5 截断和继续读取

总字符预算优先保留完整行，并返回 `next_offset`，让模型继续读取后面的行。

但有一个边界：如果第一行本身就超过总字符预算，Runtime 只能从这一行中间截断。由于 offset 是“行号”而不是“字节/字符位置”，这一行剩余的尾部无法通过下一个行 offset 找回。

因此它对正常代码和日志非常合适，但不是通用的大单行数据读取协议。

### 4.6 结构化文档与其他文件类型

Hermes 会先尝试结构化文档提取，再进入普通二进制拒绝逻辑：

~~~text
内建支持：
  .ipynb
  .docx
  .xlsx

可选文档提取器：
  .doc / .ppt / .xls
  .pptx
  OpenDocument
  RTF / EPUB / PDF 等
~~~

结构化文档的目标仍然是把源对象投影成文本 Observation，而不是把原始格式直接交给模型。

其他重要行为：

- 文档输入有大小限制，当前约 50 MB；
- PDF 主要读取文本层，扫描件或图片页会提示改用页面渲染 / OCR；
- 图片和一般二进制文件不会伪装成文本，模型应改用视觉工具或 Terminal；
- 后端采样以 Base64 穿过执行通道，避免中间 transport 提前错误解码；
- UTF-8 文本即使真的包含 `U+FFFD` 字符也不会仅凭这一点判为坏文件；
- 检测到 NUL 或无效字节序列时会按二进制处理；
- UTF-16 在一定大小范围内可通过 BOM / 字节奇偶特征救援，并警告后续编辑可能转成 UTF-8；
- 不会激进猜测各种 legacy encoding。

### 4.7 模型结果和 UI 结果并非完全相同

`read_file` Handler 最终返回 JSON 字符串，而不是 Pi 风格的独立 `content + details` 对象。

在通用 Tool Executor 中：

~~~text
原始 function_result
     ├── display callback / UI
     └── persistence / spill / aggregate budget
                    ↓
             model tool message
~~~

因此 UI callback 可以看到持久化/外溢前的结果，而模型最终收到的 Tool Message 可能已经被替换成一个结果文件引用。

Hermes 特别把 `read_file` 的单结果外溢阈值钉为无限，避免：

~~~text
read_file(A)
  ↓
结果被自动存到 B
  ↓
模型再 read_file(B)
  ↓
结果又被自动存到 C
~~~

单次读取主要依靠 `read_file` 自己的 100K 字符预算控制。不过同一轮多个工具结果加起来超过聚合预算时，仍可能发生整轮结果外溢。

---

## 5. 第三层：Cognitive / Epistemic State

这一层是 Hermes `read_file` 最有意思的地方之一。

一次读取除了产生文件系统行为，还建立了一个认知事实：

~~~text
World State：
  foo.py 当前内容为 X

Epistemic State：
  当前模型已经观察过 foo.py@X
~~~

后者不是文件系统状态，而是 Runtime 对“模型现在知道什么”的判断。

### 5.1 Dedup 不是普通 I/O Cache

Hermes 会按 Task 记录精确读取区域：

~~~text
(resolved_path, offset, limit, mtime)
~~~

相同区域的典型行为是：

~~~text
第一次读取
  → 返回完整内容

第二次读取，且 mtime 未变化
  → 返回 lightweight unchanged stub

第三次仍重复
  → BLOCKED，阻止模型陷入循环
~~~

调用其他非读取/搜索工具后，连续重复计数会被重置。不存在文件的结果还有短时间 negative cache，并会尝试给出相似文件名或修复 Unicode 等价文件名。

这套状态的目的不是节省磁盘 I/O，而是避免把模型已经拥有的 Observation 一遍遍塞回 Context。

所以它更准确的名字应当接近：

~~~text
Observed Region Tracker
~~~

而不是：

~~~text
File Content Cache
~~~

### 5.2 为什么 Context Compression 必须重置 Dedup

考虑下面的上下文：

~~~text
User
Assistant → read_file(foo.py)
Tool      → foo.py 的 1000 行内容
Assistant → 继续分析
~~~

压缩后可能变成：

~~~text
System Summary：
  Agent 检查过 foo.py，正在调查问题 X。
~~~

模型已经不再拥有那 1000 行具体内容。

如果 Runtime 的 dedup 仍然说：

~~~text
foo.py 已读，内容未变化，不再返回
~~~

就会出现状态错位：

~~~text
Runtime believes model knows X
                  ≠
Current context actually contains X
~~~

Hermes 因此会在 Context Compression 后调用 `reset_file_dedup(task_id)`，允许模型重新取得已经被压缩掉的文件内容。

### 5.3 Compression 是认知状态迁移

这个例子揭示了一个比 `read_file` 更通用的原则：

> Context Compression 不是简单地删掉几条消息，而是一次 Agent Epistemic-State Transition。

压缩前：

~~~text
模型拥有文件全文、Terminal 原始输出、搜索结果细节
~~~

压缩后：

~~~text
模型只拥有这些信息的摘要和抽象结论
~~~

模型的知识分辨率已经变化。

因此，所有依赖“模型已经看过 X”的 Runtime 状态都应该绑定 Context Lifecycle，例如：

~~~text
file read dedup
retrieval dedup
web page dedup
memory injection dedup
tool result dedup
artifact expansion state
~~~

### 5.4 Runtime 必须维护两个状态空间

从这里可以提炼出一个很重要的 Harness 设计：

~~~text
World State
  环境真实是什么样

Epistemic State
  当前模型还知道多少真实环境信息
~~~

只维护 World State 不够。Agent 的下一步决策依赖的不是整个真实世界，而是当前 Context 中仍然存在的世界投影。

---

## 6. 第四层：Consistency / Scheduling Layer

`read_file` 不仅是 Observation Tool，还是文件修改一致性协议的 read side：

~~~text
read_file
   ↓
记录 observation evidence
   ↓
file_state
   ↓
patch / write_file 提交前检查
~~~

### 6.1 这是一种“提醒型”的 Optimistic Concurrency Control

数据库里的典型过程是：

~~~text
读取 version=5
      ↓
基于该版本计算
      ↓
提交时发现 version=6
      ↓
拒绝 / 重试
~~~

Hermes 文件工具会建立类似的版本证据：

~~~text
read foo.py，记录 mtime=T1
          ↓
模型 inference、搜索、调用其他工具
          ↓
用户 / IDE / 其他 Agent 修改 foo.py，mtime=T2
          ↓
模型尝试 patch / write foo.py
          ↓
Runtime 发现读取视图已经 stale
~~~

但必须精确区分：数据库示例会拒绝旧版本提交，Hermes 当前的 `write_file` / `patch` 主要只是把 stale 信息作为警告返回，仍然会继续修改文件。

~~~text
Hermes 当前
  = 检测 stale view
  + 返回 warning
  + 仍然 Last Writer Wins
~~~

所以它已经具备 Optimistic Concurrency 的观察能力，但还不是严格的 Compare-and-Swap 提交协议。

也就是：

~~~text
Read Snapshot
      ↓
Long-running Computation
      ↓
Stale Detection
      ↓
Advisory Warning + Mutation
~~~

Agent 特别需要这种机制，因为它的 read → write 天然是一个长事务，中间可能经过数十秒、多个模型调用甚至 Subagent 协作。又不能真的持有文件锁这么久，所以 Optimistic Concurrency 比长时间悲观锁更合适。

### 6.2 `file_state` 记录什么

普通文本成功读取后，Hermes 会记录：

~~~text
task_id
resolved path
mtime / read time
是否只读取了部分内容
最近写入者
~~~

后续 mutation 可以据此判断：

- 文件是否从未被当前 Agent 读取；
- 模型是否只看过局部内容；
- 读取后文件是否被外部进程改过；
- 是否由同一进程中的兄弟 Subagent 改过；
- 当前 mutation 是否建立在过期视图上。

所以 read 的结果不只进入 Context，也会成为未来 write 的提交证据。

### 6.3 Reader / Writer Reservation 解决的是另一类问题

mtime / file state 主要处理跨工具、跨时间的 stale view。除此之外，同一批 Tool Calls 也可能产生即时竞态：

~~~json
[
  {"tool": "read_file", "path": "a.py"},
  {"tool": "write_file", "path": "a.py"}
]
~~~

如果两者直接并行，读取到旧内容还是新内容取决于调度时机。

Hermes 的 Batch Planner 会按路径建立 Reader / Writer Reservation：

~~~text
read A + read A
  → 可以并行

read A + write B
  → 路径不冲突，可以并行

read A + write A
  → 必须拆段并顺序执行

无法解析出可靠 path
  → 作为 barrier 保守处理
~~~

因此这里其实有两层一致性保护：

~~~text
同一 batch 的短时间竞态
  → Reader / Writer Scheduling

跨多个 iteration 的长时间视图过期
  → mtime + file_state Optimistic Validation
~~~

两者不能互相替代。

---

## 7. 安全边界与取消语义

### 7.1 防御性读取限制

Hermes 会拦截或特殊处理一些高风险来源，例如：

- `/dev/zero`、随机设备、stdio / tty；
- `/proc` 中的敏感端点；
- FIFO、Socket、设备文件；
- Hermes credentials、MCP token、浏览器 Profile；
- 项目中的 `.env`；
- Skills Hub cache 等。

之后还会对文本结果执行敏感信息脱敏。

但源码对这层边界的定位很诚实：它是 defense-in-depth，不是绝对安全边界。因为同一个 Agent 如果拥有 Terminal，理论上仍可能用 Shell 读取相同文件。

因此这里的真正经验是：

> Tool-local Guardrail 可以降低误触和泄露概率，但不能替代 Environment Sandbox、Capability Control 和统一 Policy。

### 7.2 Approval 与 Middleware

`read_file` 默认不是需要人工审批的危险工具，但仍会经过通用的 Tool Request Middleware、Policy / Hook 等扩展点，因此 Plugin 可以对参数进行规范化、阻止特定路径或要求审批。

安全检查必须针对 Middleware 修改后的最终参数，而不是只审批模型最初生成的参数。否则可能出现“审批 A，实际执行 B”。

### 7.3 取消不是强制终止

`read_file` 自身没有持续轮询 interrupt flag。已经进入底层执行的读取通常会继续完成。

并发执行器可以：

- 不再启动尚未开始的 future；
- 对调用方表现为超时或放弃等待；
- 阻止结果继续推进 Agent Loop。

但它不能保证杀死已经在线程中执行的底层读取。

特殊文件检查、结果预算和后端超时能降低挂死风险，但语义上仍应区分：

~~~text
停止等待 / 忽略结果
        ≠
底层 I/O 已经被物理终止
~~~

---

## 8. 当前实现中的语义裂缝

这些问题不影响我们理解整体设计，但很适合作为 Runtime Review 的案例。

### 8.1 Schema 默认值与 Handler 默认值不一致

~~~text
Schema：limit 默认 2000
Handler：缺省时使用 500
~~~

这是模型契约与真实执行行为不一致。理想上默认值应该只有一个事实来源。

### 8.2 远程文件能读，但版本证据可能退化

内容读取走 Terminal Environment，所以 Docker / SSH 等后端可以正常工作。

但部分 dedup 和 `file_state` 版本检查仍依赖宿主 Python 进程的 `os.path.getmtime(resolved_path)`。远程路径在宿主机上不存在时，mtime 证据会缺失，于是：

~~~text
Environment capability：读取成功
Consistency capability：降级或跳过
~~~

这说明抽象远程文件系统时，只统一“read bytes”还不够，还应统一 `stat/version` 语义。

### 8.3 Structured Document 提前返回，绕过统一状态主干

当前结构化文档分支在提取并脱敏后提前返回，没有完整经过普通文本路径中的：

~~~text
dedup / loop tracker
negative cache
file_state.record_read
部分读取一致性检查
~~~

于是从模型视角都是：

~~~text
read_file("foo.py")
read_file("paper.docx")
~~~

但 Runtime 内部的状态语义并不一致：普通文本读取会建立“已观察”和后续写入证据，结构化文档读取未必会。

这是一类典型 Runtime 技术债：

> Feature works locally, but violates global execution semantics.

“DOCX 能否读取”的功能测试会通过，但“DOCX 是否参与重复读取保护、Compression reset 和 stale-write evidence”可能不一致。

### 8.4 超长单行无法完整分页

分页单位是行号。单行自身超出字符预算时，行内尾部没有单独 cursor 可以继续取得。

### 8.5 可选提取器的 Lazy Install 带来不确定性

第一次遇到某些文档格式时，Runtime 可能尝试静默安装可选依赖并重试。这提高了开箱可用性，也引入了首次调用延迟、网络依赖和运行时可重复性问题。

---

## 9. 如果重构：统一 Read Pipeline

更理想的结构是让不同格式只负责解码，最终全部汇入同一条状态与预算主干：

~~~text
resolve path
    ↓
security check
    ↓
backend stat + read
    ↓
format adapter
    ├── text
    ├── utf16
    ├── ipynb
    ├── docx / xlsx
    └── pdf / other extractors
    ↓
NormalizedDocument
    ↓
pagination
    ↓
line shaping
    ↓
budget enforcement
    ↓
sanitization
    ↓
dedup / loop detection
    ↓
record observation state
    ↓
Tool Result
~~~

`NormalizedDocument` 可以至少包含：

~~~text
text
total_lines
source_identity
source_version
projection_identity
projection_version
truncated
next_offset
~~~

这样 TXT、DOCX、PDF、XLSX 只是不同 Decoder，而不再拥有不同的 `read_file` 生命周期语义。

### 9.1 Derived Representation 的版本证据

对于普通文本，版本证据近似是：

~~~text
path + mtime
~~~

但对于 PDF / DOCX / XLSX，模型看到的是经过提取器生成的派生表示。更完整的 Observation Version 应该是：

~~~text
ObservationVersion
  = SourceVersion
  + ProjectionVersion
~~~

例如：

~~~text
paper.pdf
source mtime = 12345
extractor = anydoc-v2
page scope = all
projection options = ...
~~~

即使源 PDF 没变，提取器升级后模型看到的文本也可能发生变化。因此“模型读过什么”应同时绑定源文件版本和投影算法版本。

---

## 10. 与其他 Harness 比较时应看什么

后续与 Pi 或其他 Harness 比较，不要只比较参数名。更值得比较的是：

1. 文件工具和 Terminal 是否绑定同一个 Environment；
2. offset 是行、字节还是字符，截断后的内容能否完整找回；
3. 图片是由 read 直接内联，还是交给独立 Vision Tool；
4. Tool Result 是否区分模型内容与 UI details；
5. 单行、单结果、单轮、总 Context 是否有分层预算；
6. repeated read 是普通缓存，还是与 Context Lifecycle 绑定的认知状态；
7. Compression 后是否重置“模型已看过”的状态；
8. read 是否为 patch / write 建立 stale-view evidence；
9. 同批 read / write 是否具有资源冲突调度；
10. 本地和远程 Environment 是否同时具备 read、stat、version 和 cancellation 语义。

---

## 11. 最终记忆模型

不用记住所有实现细节，记住四句话即可：

~~~text
1. Environment Binding
   read_file 必须读取 Agent 当前正在工作的那个世界。

2. Observation Shaping
   真实文件不能原样塞给模型，要转换成有预算、可定位的 Observation。

3. Epistemic Bookkeeping
   Runtime 必须知道模型现在看过什么，且该状态必须随 Context 生命周期变化。

4. Consistency
   read 建立 observation evidence，write 会检查 evidence；
   但 Hermes 当前只警告 stale view，不会拒绝覆盖；
   并发执行再用 reader / writer 调度避免物理竞态。
~~~

完整关系可以压成：

~~~text
                 ┌──────── Environment Binding
                 │
Filesystem ──► Read Adapter
                 │
                 ├──────── Observation Budget
                 │
                 ▼
            Model Context
                 │
                 ├──────── Epistemic State
                 │
                 ▼
          Future Decisions
                 │
                 ▼
          patch / write_file
                 │
                 └──────── Version Check + Warning
~~~

最终可推广到整个 Agent Runtime 的原则是：

> Runtime 不能只维护“世界现在是什么”，还必须维护“模型当前还知道多少世界状态”。

## 12. 主要代码入口

~~~text
tools/file_tools.py
  READ_FILE_SCHEMA
  _handle_read_file()
  structured document routing
  char budget / redaction / dedup integration

tools/file_operations.py
  ShellFileOperations
  environment-bound read
  line pagination / line shaping / encoding detection

tools/read_extract.py
  structured document extraction

tools/file_state.py
  read evidence and stale-view tracking

tools/budget_config.py
  per-result and per-turn tool-output budget policy

tools/tool_result_storage.py
  result persistence / spill

agent/tool_dispatch_helpers.py
  reader / writer resource planning

agent/conversation_compression.py
  compression-time file dedup reset

agent/file_safety.py
  sensitive-path and special-file guardrails
~~~
