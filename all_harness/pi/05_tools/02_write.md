# Pi Concrete Tool 02：write

> 状态：已共同阅读并确认。

## 1. 工具定位

`write` 是 Pi 文件工具中最直接的写入原语：模型提供文件的完整最终内容，Harness 创建或整体覆盖目标文件。

~~~text
模型声明完整文件状态
        ↓
创建父目录
        ↓
整体写入目标文件
~~~

它不是局部修改工具，也不是 patch 工具。

## 2. 模型可见接口

~~~typescript
write({
  path: string,
  content: string
})
~~~

- `path`：相对路径或绝对路径；
- `content`：文件完整文本内容。

System Prompt 另外加入：

~~~text
Create or overwrite files
Use write only for new files or complete rewrites.
~~~

因此 Pi 对模型约定：

~~~text
创建新文件       → write
彻底重写旧文件   → write
修改已有文件几行 → edit
~~~

## 3. 内部执行流程

~~~text
write(path, content)
        ↓
resolveToCwd(path, cwd)
        ↓
withFileMutationQueue(absolutePath)
        ↓
operations.mkdir(parent, recursive=true)
        ↓
operations.writeFile(path, content)
        ↓
Successfully wrote ... bytes
~~~

默认后端执行：

~~~typescript
fs.mkdir(parentDirectory, { recursive: true })
fs.writeFile(absolutePath, content, "utf-8")
~~~

文件不存在时创建；文件存在时整体覆盖。

## 4. write 与 edit 的职责边界

`write` 表达的是完整目标状态：

~~~text
setFileContent(path, completeContent)
~~~

`edit` 表达的是已有文件中的局部变化：

~~~text
replace(oldText, newText)
~~~

新文件没有可供 `edit` 匹配的 `oldText`，完整重写也没有必要构造多组局部替换。因此两者不是重复工具。

但对已有大文件使用 `write` 风险较高：模型必须重新生成整个文件，任何遗漏都会被真实覆盖。Pi 主要通过 Prompt guideline 约束模型只在新建文件或完整重写时使用它。

## 5. write 不做什么

执行前，`write` 不会：

- 读取旧文件；
- 检查旧文件内容；
- 比较文件版本或 hash；
- 生成 diff；
- 自动备份或创建 checkpoint；
- 写入后重新读取并校验；
- 判断整体覆盖是否符合用户意图。

所以原生 `write` 不是安全文件编辑器，而是一个完整文件状态写入原语。Approval、备份、checkpoint 或额外权限策略如果存在，需要由 Tool System Hook、Extension 或外围执行环境提供。

## 6. 同文件修改队列

`write` 和 `edit` 共享 `withFileMutationQueue()`：

~~~text
write A(file.ts)
        ↓
write B(file.ts)
~~~

同一路径的修改串行执行，不同文件仍可以并行。已经存在的符号链接会通过 `realpath()` 归一化，因此指向同一目标文件的别名会进入同一队列。

~~~text
target.ts
alias.ts → target.ts

write(target.ts)
        ↓
edit(alias.ts)
~~~

这个队列解决的是同一 Pi 进程内部修改操作交错的问题，不解决：

- 外部编辑器或其他进程同时修改；
- 多个 Pi 进程之间的竞争；
- `read` 与 `write` 同批并发时的读取竞态；
- 后一个完整写入在顺序上覆盖前一个修改。

队列保证物理执行顺序，不保证修改语义正确。

## 7. 取消语义：Abort 不等于回滚

`write` 会在创建目录和写入操作前后检查 `AbortSignal`，但默认 `fs.writeFile()` 一旦开始，并不会被真正强制终止。

可能发生：

~~~text
writeFile 开始
        ↓
用户触发 abort
        ↓
底层写入仍然完成
        ↓
Harness 检测到 aborted
        ↓
工具返回 Operation aborted
~~~

因此工具报告取消，不代表文件一定没有变化。

Pi 不会在 abort 事件发生时立即释放修改队列。它会等待正在进行的底层操作结束，再让下一次同文件修改开始，避免迟到的第一次写入和第二次写入交错。

测试明确验证：被取消的写操作可以先完成副作用，工具 Promise 随后报错，下一次写入最后再执行。

准确语义是：

~~~text
Abort
  = 停止继续接受本次操作结果
  + 等待在途副作用结束后释放队列
  ≠ 撤销已经发生的文件修改
~~~

## 8. 写入不是事务或原子替换

默认实现直接调用 `fs.writeFile()`，没有采用：

~~~text
写临时文件
  ↓
fsync
  ↓
rename 原子替换
~~~

因此如果进程或底层文件系统在覆盖过程中失败，原生工具不提供事务性恢复保证，也不会自动恢复旧内容。

`fileMutationQueue` 只能阻止 Pi 内部的写操作互相交错，不能把单次写入变成事务。

## 9. WriteOperations：工具语义与后端分离

底层写入动作被抽象为：

~~~typescript
interface WriteOperations {
  writeFile(absolutePath, content): Promise<void>
  mkdir(directory): Promise<void>
}
~~~

默认绑定本机文件系统。Harness、SDK 或 Extension 可以在创建工具时注入其他实现：

~~~typescript
createWriteTool(cwd, {
  operations: someWriteOperations
})
~~~

这与 `read` 的边界相同：

> Operations 是宿主侧依赖注入，不是模型通过 `path` 动态选择后端。

模型接口中不存在 `host`、`containerId`、`backend` 或 `transport`。

### 可选 SSH Extension

Pi 仓库的 SSH 示例会把 `write` 重绑定到指定远程主机：

~~~text
content
  ↓ Base64 编码
ssh user@host
  ↓
echo <base64> | base64 -d > remotePath
~~~

父目录通过远程 `mkdir -p` 创建。模型仍然调用 `write(path, content)`，实际后端由 Extension 提前决定。

## 10. 文本能力边界

`content` 只能是字符串，默认以 UTF-8 写入。原生接口没有：

- Buffer 或二进制内容；
- Base64 模式；
- MIME 类型；
- append 模式；
- create-only / fail-if-exists 模式；
- 文件权限或 executable bit 参数；
- 编码参数。

因此 `write` 是文本文件写入工具，不是完整文件系统 API。

## 11. Tool Result 与 Context

成功后只返回：

~~~text
Successfully wrote N bytes to <path>
~~~

不会把完整文件内容再次放进 Tool Result。因为完整 `content` 已经存在于 assistant 的 Tool Call 参数中，再返回一次只会重复占用 Context。

结果的 `details` 为 `undefined`，因为 `write` 不生成 diff 或其他结构化数据。失败时，Agent Loop 会把异常转换成错误 Tool Result 交给模型。

## 12. TUI 展示

模型流式生成 `content` 参数时，Pi 会增量预览写入内容，并根据文件扩展名做语法高亮。

~~~text
模型生成 write 参数
        ↓
TUI 增量展示内容
        ↓
参数完整
        ↓
工具执行
~~~

折叠状态显示前 10 行，展开后显示全部。这是可观察性能力，不是审批流程；TUI 展示内容不会自动暂停并等待用户确认。

## 13. 成功消息中的 bytes 不准确

成功消息使用：

~~~typescript
content.length
~~~

但 JavaScript 字符串长度不是 UTF-8 字节数。例如：

~~~typescript
"你好".length === 2
Buffer.byteLength("你好", "utf-8") === 6
~~~

因此非 ASCII 内容对应的 `N bytes` 实际是 UTF-16 code unit 数量，不是真实写入字节数。这只影响结果描述，不影响文件内容。

## 14. 设计取舍

### 优点

- 模型接口只有 `path` 和 `content`；
- 新建文件和完整重写的语义直接；
- 自动递归创建父目录；
- 同文件修改不会在 Pi 进程内交错；
- Operations 可以替换为远程或隔离环境后端；
- 成功结果不重复回传完整内容。

### 代价和限制

- 覆盖已有文件没有内容前置条件；
- 没有 diff、备份、checkpoint 和写后验证；
- 不是原子写入或事务；
- Abort 不保证撤销副作用；
- 只支持文本；
- 路径不是 cwd 权限沙箱；
- 安全性主要依赖模型遵守 Prompt 以及外围 Hook 和执行环境。

## 15. 代码证据

~~~text
模型 Schema、Prompt、执行和 TUI：
packages/coding-agent/src/core/tools/write.ts

同文件修改队列：
packages/coding-agent/src/core/tools/file-mutation-queue.ts

并发与取消测试：
packages/coding-agent/test/file-mutation-queue.test.ts

工具集合：
packages/coding-agent/src/core/tools/index.ts

可选 SSH 后端示例：
packages/coding-agent/examples/extensions/ssh.ts

可选 Gondolin VM 后端示例：
packages/coding-agent/examples/extensions/gondolin/index.ts
~~~

