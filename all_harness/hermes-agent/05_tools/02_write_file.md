# Hermes Core Tool 02：write_file

> 状态：已共同阅读。
>
> 一句话定位：模型声明“整个文件最终应该是什么”，Hermes 负责安全地把这个目标状态提交到文件系统。

## 1. 先记住四句话

~~~text
1. write_file 是整个文件覆盖，不是局部编辑。

2. 写之前会检查路径和内容，防止明显的数据损坏。

3. 它先写临时文件，再一次性替换，避免留下半个文件。

4. 它能发现“你读完后别人改过”，
   但目前只警告，仍然会覆盖。
~~~

第四点最关键：Hermes 有版本冲突意识，但当前还是“提醒型保护”，不是“强制型保护”。

## 2. 它和 `patch` 的区别

`write_file` 接收完整文件内容：

~~~text
无论原文件是什么
        ↓
整个替换成模型提交的 content
~~~

它没有 append、old_content 或 expected_version。

~~~text
创建新文件 / 完整重写
  → write_file

修改已有文件的一小部分
  → patch
~~~

`write_file` 不返回 diff；`patch` 才返回修改前后的 diff。

## 3. 整体流程

~~~text
模型提交 path + 完整 content
              ↓
参数是否完整
              ↓
路径是否允许写
              ↓
内容是否明显有问题
              ↓
写入临时文件
              ↓
一次性替换目标文件
              ↓
Hash / Lint / LSP 验证
              ↓
更新 Runtime 文件状态
~~~

### 3.1 缺失 content 不等于空 content

Hermes 明确区分：

~~~json
{"path": "foo.py", "content": ""}
~~~

这表示真的要清空文件，是合法操作。

~~~json
{"path": "foo.py"}
~~~

这可能是模型在上下文压力下丢失了 Tool Call 参数，Hermes 会拒绝执行并要求重发。

否则一次参数截断就可能被误解成“把文件清空”。

## 4. 写入前会拦什么

### 4.1 路径保护

硬拒绝的典型目标：

~~~text
系统敏感路径
Hermes 安全配置
SSH 私钥和 authorized_keys
配置的 Safe Root 之外
~~~

需要人工批准的典型目标：

~~~text
AGENTS.md / CLAUDE.md / SOUL.md / .cursorrules
~/.ssh/config
~~~

前一组会影响未来 Agent 行为，是持久化 Prompt Injection 入口；后一项可以通过 SSH 配置间接执行命令。

修改其他 Hermes Profile 的 skills、plugins、cron、memories 默认也会被拦截。用户明确要求后，可用 `cross_profile=true` 绕过这层软保护。

### 4.2 防止把 `read_file` 的显示内容写回文件

`read_file` 返回：

~~~text
1|import os
2|print("hello")
~~~

模型有时会把行号一起交给 `write_file`。Hermes 会识别大量连续的 `LINE_NUM|CONTENT` 并拒绝写入。

它也会拒绝把下面这种 Runtime 提示写入真实文件：

~~~text
File unchanged since last read...
~~~

本质是在防止：

~~~text
模型看到的 Observation
        ↓ 被误认为文件内容
污染真实 World State
~~~

### 4.3 防止把提取文本写回 DOCX / XLSX / PDF

`read_file` 给模型看的 DOCX、XLSX、PDF 内容是提取后的文本，不是原始文档字节。

如果把提取文本写回 `.docx`，文档会被摧毁成普通文本。因此这些容器格式禁止用 `write_file` 写入；已有 PDF 也不能被普通文本覆盖。

### 4.4 内容语法检查

对于 JSON、YAML、TOML：

~~~text
候选内容无法解析
  → 写入前拒绝
  → 原文件不变
~~~

Python 不同：

~~~text
即使存在 SyntaxError
  → 仍然写入
  → 在结果中报告错误
~~~

所以：

> 文件成功落盘，不等于写进去的代码正确。

## 5. 为什么不容易留下“半个文件”

Hermes 不直接截空目标文件，而是：

~~~text
在目标旁创建临时文件
          ↓
写完全部内容
          ↓
一次性替换目标文件
~~~

写到一半失败时，原文件通常仍然完整；其他 Reader 也不会看到半截内容。

它还会尽量保留：

~~~text
原文件权限
CRLF / LF 换行风格
UTF-8 BOM
Symlink 本身
~~~

父目录不存在时会自动创建。内容通过 stdin 传输，避免 Shell 参数长度和转义问题。

但这里的“原子”只表示不会暴露半截文件，不代表完整数据库事务：替换完成后没有自动 rollback，也不是严格的断电恢复协议。

## 6. 写完以后做什么

### 6.1 Hash 验证

Hermes 比较：

~~~text
准备写入内容的 Hash
        vs
磁盘真实内容的 Hash
~~~

一致时返回：

~~~json
{"verified": true}
~~~

这告诉模型文件已经确认落盘，不需要马上再 `read_file` 一次。

Hash 不一致会返回硬错误，但这是写后检测，不会自动恢复旧文件。

### 6.2 Lint 和 LSP

之后还会执行：

~~~text
语法 / Lint
      ↓
可选 LSP 语义诊断
~~~

Hermes 尽量只返回本次写入新引入的问题，避免把早已存在的错误重复塞回 Context。

### 6.3 更新模型认知状态

成功写入意味着模型刚刚提交了完整内容，因此 Runtime 认为模型知道文件的新状态。

它会：

~~~text
记录当前 Task 是最后写入者
更新文件版本记录
清除该路径的旧 read dedup
清除旧的 not-found cache
把之前的测试/验证结果标记为 stale
~~~

旧 dedup 必须清除，因为它对应的是写入前的文件。

## 7. 最大的问题：发现冲突后仍然会覆盖

~~~text
Agent A 读取版本 1
        ↓
Agent B 写成版本 2
        ↓
Agent A 根据旧版本写入版本 3
~~~

Hermes 能发现 B 在 A 读取后修改过文件，但当前行为是：

~~~text
让 A 覆盖成功
      ↓
再返回 stale warning
~~~

所以它防止的是“无声覆盖”，不是覆盖本身。

~~~text
Hermes 当前
  = Last Writer Wins
  + Stale-write Warning
~~~

严格保护应该是：

~~~text
发现文件版本变化
      ↓
拒绝本次写入
      ↓
要求重新读取
      ↓
基于新版本再次修改
~~~

因此，`read_file` 虽然建立了版本证据，`write_file` 当前并不强制要求这份证据有效。

### 并发锁也不能解决这个问题

Hermes 会让同一路径的 read/write、write/write 顺序执行，也会给同进程 Subagent 使用 per-path lock。

这只能保证两个写入流程不互相穿插，不能阻止后一个 Writer 覆盖前一个 Writer。

~~~text
Path Lock
  → 防执行竞态

Version Check + Reject
  → 防基于旧内容覆盖新内容
~~~

Hermes 做好了第一层，第二层目前只有警告。

## 8. 中断、恢复与模型虚报

`write_file` 是有副作用的工具。中断时可能：

~~~text
最终替换前中断
  → 原文件通常还在

最终替换后中断
  → 新内容已经落盘

落盘后、Tool Result 保存前崩溃
  → Runtime 不知道是否完成
~~~

恢复 Session 时，Hermes 会把最后一种情况标记为：

~~~text
effect UNKNOWN
先检查真实文件状态，再决定是否重试
~~~

它不会盲目重放副作用工具。

另外，如果写入失败但模型最终声称“文件已经修改”，Runtime 会追加 File Mutation Verifier 警告，列出实际没有修改成功的文件。

它会正确区分：

~~~text
文件已写入，但 lint 报错
  → mutation 已发生

文件写入本身失败
  → mutation 未发生
~~~

## 9. 最终记忆模型

~~~text
模型声明完整文件状态
        ↓
检查路径和候选内容
        ↓
临时文件原子替换
        ↓
Hash / Lint / LSP 建立结果证据
        ↓
更新模型认知和最后写入者状态
~~~

最值得记住的五个判断：

~~~text
缺失 content 不等于空 content
Observation 不能直接写回 World State
原子落盘不等于代码正确
执行顺序安全不等于没有 Lost Update
工具中断不等于副作用没有发生
~~~

主要代码入口：

~~~text
tools/file_tools.py
tools/file_operations.py
tools/file_state.py
agent/tool_dispatch_helpers.py
agent/tool_result_classification.py
agent/turn_finalizer.py
agent/replay_cleanup.py
~~~
