# Hermes Core Tool 05：search_files

> 状态：已共同阅读。
>
> 一句话定位：`search_files` 把“搜文件”和“搜文件内容”合并成一个模型工具，通过参数切换能力，负责先把未知文件系统缩小成候选范围。

## 1. 先记住三句话

~~~text
1. target=files
   按文件名 / glob 搜索文件，接近 find 和部分 ls。

2. target=content
   按 regex 搜索文件内容，接近 rg / grep。

3. search_files 只负责发现候选；
   找到目标后，仍然应该用 read_file 获取完整证据。
~~~

所以整体协作关系是：

~~~text
不知道代码在哪里
        ↓
search_files 缩小范围
        ↓
得到候选文件 / 匹配行
        ↓
read_file 读取完整上下文
        ↓
patch / write_file 修改
~~~

## 2. 一个工具，两种搜索

Hermes 没有分别暴露 `grep`、`find` 和 `ls`，而是用 `target` 参数切换搜索模式。

### 2.1 搜文件内容：`target=content`

~~~json
{
  "pattern": "ProviderProfile",
  "target": "content",
  "path": "agent",
  "file_glob": "*.py"
}
~~~

这条调用表示：

> 在 `agent` 目录下的 Python 文件中，搜索匹配 `ProviderProfile` 的内容。

它支持：

- regex；
- `file_glob` 文件类型过滤；
- 匹配行前后的 `context`；
- `content`、`files_only`、`count` 三种输出模式；
- 使用 `offset + limit` 分页。

当 regex 明确包含换行时，Hermes 会自动开启 ripgrep 的 multiline 模式，而不是让模型自己猜应该添加什么命令参数。

### 2.2 搜文件名：`target=files`

~~~json
{
  "pattern": "*provider*.py",
  "target": "files",
  "path": "agent"
}
~~~

这条调用表示：

> 在 `agent` 目录下，寻找文件名符合 `*provider*.py` 的文件。

结果按修改时间排序，最近修改的文件优先出现。

它只承担一部分 `ls` 能力：模型可以用它发现目录下有哪些目标文件，但它不是完整的目录浏览器，不重点返回权限、文件大小、所有子目录和详细时间等信息。

## 3. 为什么合并成一个工具

如果拆开提供：

~~~text
grep
rg
find
ls
~~~

模型每次请求都要接收多份 Tool Schema，还要先判断该选哪个名字。

Hermes 把它们压缩成：

~~~text
search_files
  ├── target=content
  └── target=files
~~~

收益是：

- 核心 Tool Schema 更少；
- 文件发现只有一个统一入口；
- cwd、远程环境、路径安全、输出预算和重复搜索保护可以复用同一套协议；
- 更符合 Hermes 的 narrow waist：核心入口少，参数表达变化。

代价也很直接：一个 Schema 里的参数分支更多。例如 `file_glob`、`context` 和 `output_mode` 主要服务内容搜索，并不是所有模式都同样有意义。

## 4. 它不是简单执行一条 `rg`

实际流程大致是：

~~~text
模型提交搜索参数
        ↓
规范化 target / offset / limit
        ↓
绑定当前 Task 的 cwd 与 Terminal Environment
        ↓
检查路径是否允许读取
        ↓
检查是否在重复相同搜索
        ↓
选择 rg / grep / find
        ↓
过滤敏感结果并脱敏内容
        ↓
压缩、限量并返回 Tool Result
~~~

因此，它搜索的是当前 Agent 所在的环境，而不一定是 Hermes 主进程本机：

~~~text
Local Task  → 搜本机文件
Docker Task → 搜容器文件
SSH Task    → 搜远程主机文件
~~~

这和 `read_file`、`write_file`、`terminal` 必须看到同一个世界，是同一条原则。

## 5. 搜不到时，它会帮模型判断原因

普通搜索工具可能只返回：

~~~json
{"total_count": 0}
~~~

Hermes 会额外做少量、受限的探测。

### 5.1 大小写可能写错

如果精确搜索为零，但忽略大小写后能搜到，它会提示：

~~~text
可能是 pattern 的大小写不对，相关结果位于这些文件……
~~~

### 5.2 正则符号可能被误解

例如模型想搜索字面量：

~~~text
lookup[key+1]
~~~

但 `[]`、`+` 在 regex 中有特殊含义。Hermes 会再做一次固定字符串探测，并提示模型可能需要转义。

### 5.3 结果可能只在隐藏或 Git Ignore 文件中

Hermes 默认不返回隐藏目录和 `.gitignore` 排除的文件。如果普通结果为零，它会探测这些位置是否存在匹配，并告诉模型：

~~~text
结果存在，但位于默认排除的隐藏 / ignored 文件中。
~~~

它只给提示，不会悄悄把这些内容全部注入 Context。

### 5.4 Path 可能写错或混入了多个路径

单个路径不存在时，Hermes 会寻找附近名字相似的路径。

如果模型错误地传入：

~~~text
agent tools tests
~~~

Hermes 会尝试把它拆成多个路径，搜索实际存在的部分，并说明跳过了哪些路径。

这些机制本质上是在减少弱模型的搜索空转：它不替模型得出答案，但会给出下一步修正方向。

## 6. 返回结果如何控制 Context 成本

默认只返回 50 条结果，并支持：

~~~text
limit  = 本次最多取多少条
offset = 跳过前面多少条
~~~

如果还有结果，会明确提示模型使用新的 `offset` 继续读取。

匹配数量较少时，返回结构化数组：

~~~json
{
  "matches": [
    {"path": "agent/a.py", "line": 42, "content": "class ProviderProfile:"}
  ]
}
~~~

匹配较多时，会自动按文件分组：

~~~text
agent/a.py
  42: class ProviderProfile:
  76: profile = ProviderProfile(...)

agent/b.py
  18: def load_provider_profile(...):
~~~

这样不用为每条匹配重复输出完整路径和 JSON 字段，同时仍然保留文件名、行号和原始缩进。

## 7. 为什么默认排除隐藏目录

这不只是性能优化，也是 Context 安全边界。

Hermes 曾遇到隐藏缓存目录中存在来自社区内容的恶意文字，搜索结果把这些文字送进模型后，形成间接 Prompt Injection。

所以默认行为是：

~~~text
普通搜索
  → 遵守 .gitignore
  → 跳过隐藏目录

用户明确指定隐藏目录
  → 可以搜索该目录
~~~

macOS 上从过大的父目录开始搜索时，还会跳过受 TCC 保护的 Desktop、Documents、Downloads 等目录，避免无人值守搜索突然触发系统隐私弹窗。明确搜索目标目录时则允许正常访问。

除此之外，Wrapper 还会：

- 阻止读取明确敏感的路径；
- 从结果中去掉凭证或 secret 文件；
- 对匹配文本里的敏感信息做脱敏；
- 告诉模型有多少结果因为安全原因被省略。

## 8. 重复搜索保护

Hermes 会记录连续完全相同的搜索：

~~~text
第 1 次：正常执行
第 2 次：正常执行
第 3 次：警告不要重复
第 4 次及以后：阻止执行，要求根据已有结果继续
~~~

“完全相同”包括：

~~~text
pattern
target
path
file_glob
limit
offset
~~~

因此，修改 `offset` 继续翻页不算重复搜索。

不存在的搜索根目录也会进入一个短期负缓存，避免模型反复触发昂贵的远程文件系统探测；目录后来被创建后，缓存会失效。

## 9. 它是只读工具，但仍参与并发调度

`search_files` 不修改文件，因此通常属于 safe read tool，可以和 `read_file` 或另一个 `search_files` 并行。

但如果同一批 Tool Call 中出现：

~~~text
search_files(path="src")
write_file(path="src/a.py")
~~~

两者路径重叠，Hermes 会把它们拆到不同的顺序 Segment 中，避免搜索恰好观察到写入一半或前后不确定的状态。

~~~text
search A + read A
  → 可以并行

search A + write A
  → 顺序执行

search A + write B
  → 路径不重叠时可以并行
~~~

所以“只读”表示它不制造文件副作用，不表示它完全不消耗资源，也不表示它能无条件和所有工具并行。

## 10. 与 Pi 的差异

Pi 更接近把基础原语直接交给模型：

~~~text
grep
find
ls
~~~

Hermes 则先抽象出共同目的：

~~~text
从文件系统中发现候选对象
~~~

然后用一个 `search_files` 表达不同搜索方式。

可以把区别记成：

~~~text
Pi
  = 命令式原语更清晰
  = 每个工具职责更单一

Hermes
  = 模型入口更少
  = Runtime 可以统一处理搜索恢复、预算、安全和并发
  = 单个 Schema 的参数分支更多
~~~

## 11. 最终结论

`search_files` 最准确的定位不是“封装 ripgrep”，而是：

> **Filesystem → Candidate Set 的受控发现器。**

它完成的是：

~~~text
未知文件系统
      ↓
受预算和安全约束的搜索
      ↓
少量候选文件 / 匹配行
      ↓
交给 read_file 精读
~~~

最值得记住的设计是：

~~~text
一个入口，两种搜索
        +
搜不到时主动提供修正线索
        +
隐藏内容默认不进入 Context
        +
搜索结果按模型消费成本塑形
        +
与写操作共享路径冲突调度
~~~

## 12. 主要代码入口

~~~text
tools/file_tools.py
  search_tool()
  SEARCH_FILES_SCHEMA

tools/file_operations.py
  SearchResult
  search()
  _search_files()
  _search_content()
  _zero_match_probe()

agent/tool_dispatch_helpers.py
  search_files 的路径级 Reader Reservation
~~~
