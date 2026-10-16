# Pi Concrete Tool 03：edit

> 状态：已共同阅读并确认。

## 1. 工具定位

`edit` 是一个带内容前置条件的批量文本替换工具。

模型不能只指定不稳定的行号，而必须提供目标文件当前应该包含的 `oldText`。Harness 只有确认旧内容存在、唯一且不与其他修改重叠后，才会替换成 `newText`。

~~~text
模型声明修改前提和目标内容
        ↓
Harness 读取当前文件并验证
        ↓
统一应用全部修改
        ↓
写回并生成 diff
~~~

## 2. 模型可见接口

~~~typescript
edit({
  path: string,
  edits: [
    {
      oldText: string,
      newText: string
    }
  ]
})
~~~

虽然顶层只有 `path` 和 `edits` 两个字段，但 `edits[]` 可以在同一次 Tool Call 中表达多个不相交的修改。

System Prompt 会告诉模型：

- `oldText` 必须匹配目标文件；
- 同一文件的多个分离修改应放进一个 `edits[]`；
- 每个 `oldText` 都匹配原始文件，而不是前一个 edit 修改后的内容；
- 不允许重叠或嵌套 edit；
- `oldText` 在保持唯一的前提下应尽可能小。

## 3. 为什么使用 oldText，而不是行号

行号会随着其他修改发生变化：

~~~text
模型读取时：目标在第 20 行
其他修改插入 5 行
执行时：目标已经在第 25 行
~~~

`oldText` 相当于文本级修改前置条件：

~~~text
执行时仍然存在模型看到的旧内容
        ↓ 是
允许替换

执行时旧内容不存在或不唯一
        ↓
拒绝修改
~~~

它类似一种乐观锁，但没有显式文件版本号或 hash，因此不构成完整并发控制。

## 4. 内部执行流程

~~~text
edit(path, edits[])
        ↓
prepareArguments()
        ↓
验证 edits 非空
        ↓
resolveToCwd(path, cwd)
        ↓
withFileMutationQueue(path)
        ↓
检查文件可读写
        ↓
读取当前文件
        ↓
分离 UTF-8 BOM
        ↓
检测并统一换行符为 LF
        ↓
匹配所有 oldText
        ↓
验证存在、唯一、不重叠、有实际变化
        ↓
从后向前应用替换
        ↓
恢复原始换行符和 BOM
        ↓
整体写回文件
        ↓
生成 display diff、unified patch、firstChangedLine
~~~

因此它底层仍然是：

~~~text
read → modify in memory → write whole file
~~~

不是操作系统层面的局部 patch。

## 5. 多个 edits 全部匹配原始文件

假设原文件是：

~~~text
foo
bar
baz
~~~

模型提交：

~~~json
{
  "edits": [
    {"oldText": "foo\n", "newText": "foo bar\n"},
    {"oldText": "bar\n", "newText": "BAR\n"}
  ]
}
~~~

第二个 `oldText` 匹配的是原文件中原本存在的 `bar`，不是第一个 edit 新插入的 `bar`。

~~~text
原始文件
  ├── 定位 edit[0]
  └── 定位 edit[1]
        ↓
全部验证完成
        ↓
按位置从后向前统一应用
~~~

这样可以避免 edits 因数组顺序不同而产生不同语义，也避免前面的替换改变后面修改的字符偏移量。

## 6. 单次调用的逻辑原子性

写入前，每个 edit 都必须通过验证：

- `oldText` 不能为空；
- `oldText` 必须能找到；
- 每段 `oldText` 只能匹配一次；
- 多个修改区域不能重叠；
- 替换后必须产生实际变化。

只要一个 edit 失败，整个调用不会写文件：

~~~text
edit[0] 能找到
edit[1] 找不到
        ↓
整个调用失败
        ↓
edit[0] 也不会应用
~~~

这提供的是写入前的逻辑原子性。最终仍然使用普通 `writeFile()` 整体写回，没有临时文件、原子 rename 或失败回滚，因此不具备文件系统事务性。

## 7. 唯一匹配与重叠检查

如果同一个 `oldText` 出现多次，Pi 不会随机选择：

~~~text
return result;
...
return result;
~~~

模型必须扩大上下文，让目标片段在文件中唯一。但 `oldText` 也不应携带大量无关内容，因为文本越长，文件稍有变化就越容易匹配失败，也会增加 Tool Call 的 token 成本。

多个 edits 如果覆盖同一段区域也会失败：

~~~text
edit[0]: one + two
edit[1]:       two + three
               ↑ overlap
~~~

模型应把相邻或重叠修改合并成一个 edit。

## 8. 严格模型契约与有限模糊容错

Pi 对模型公开的契约要求精确匹配，但执行器在完全匹配失败后会尝试 fuzzy matching：

~~~text
模型侧契约：严格
执行器内部：有限容错
~~~

模糊规范化包括：

- Unicode NFKC；
- 删除每行末尾空格；
- 智能引号转换为普通引号；
- Unicode 横线转换为 ASCII `-`；
- 特殊空格转换为普通空格；
- 全角兼容字符转换为标准形式。

例如文件中的：

~~~typescript
console.log(‘hello’);
~~~

可以被模型提供的下面文本匹配：

~~~typescript
console.log('hello');
~~~

模糊匹配后仍然要求唯一。如果多个片段规范化后相同，工具会按重复匹配拒绝修改。

## 9. 模糊匹配的隐藏代价

如果一次调用中的任何 edit 需要 fuzzy matching，Pi 会在规范化后的内容空间中处理这一批修改。

为了避免整个文件被重写成规范化形式，它会复制未触碰行的原始内容，只把修改涉及的行从规范化内容写回：

~~~text
未修改行       → 保留原始字节
修改涉及的行   → 在规范化内容上应用替换
~~~

因此远处未修改行的尾部空格和 Unicode 形式可以保留。但修改所涉及的行中，即使某些字符不属于主要变化，也可能因为 Unicode 或尾部空格规范化而改变。

fuzzy matching 是实用容错，不是完全无损匹配。

## 10. BOM 与换行符保护

模型通常生成 LF 文本，但目标文件可能使用 CRLF。Pi 会：

~~~text
读取原文件
  ↓
取出 UTF-8 BOM
  ↓
检测 CRLF / LF
  ↓
内部统一成 LF 后匹配
  ↓
应用修改
  ↓
恢复原来的换行形式
  ↓
重新加回 BOM
~~~

因此修改 Windows 文本时，不会因为修改一处就把整个文件的 CRLF 改成 LF。

对于本身混合多种换行符的文件，Pi 根据最先检测到的换行形式恢复，可能把混合文件统一成一种换行格式。

## 11. 模型参数兼容层

公开 Schema 始终保持为 `path + edits[]`，但 `prepareArguments()` 会在 Schema 校验前修复常见模型输出错误：

~~~text
edits 是 JSON 字符串
  → 尝试解析成数组

edits 是单个对象
  → 包装成数组

旧版顶层 oldText / newText
  → 合并进 edits[]
~~~

这体现了一个典型 Harness 模式：

> 对模型公开一个干净、严格的新协议，在执行边界对已知模型错误和旧接口做兼容。

无效 JSON 不会被猜测性修复，后续 Schema 校验会正常拒绝。

## 12. Diff 与 Tool Result 分层

执行成功后返回：

~~~typescript
details: {
  diff,
  patch,
  firstChangedLine
}
~~~

- `diff`：面向 TUI 展示，带行号，只保留修改附近的上下文；
- `patch`：标准 unified patch，可以被 patch 工具重新应用；
- `firstChangedLine`：供编辑器定位第一次修改。

多个修改距离很远时，display diff 会折叠中间的大段不变内容。

模型进入下一轮 Context 的 `content` 只包含：

~~~text
Successfully replaced N block(s) in <path>.
~~~

完整 diff 和 patch 保存在 `details` 中，主要供 UI、日志和宿主逻辑使用。模型已经在自己的 Tool Call 参数中保存了计划修改，不需要再次接收完整 diff。

## 13. TUI 预览不是执行结果

模型生成完整参数后，TUI 会先调用 `computeEditsDiff()`：

~~~text
Tool Call 参数完整
        ↓
读取文件并计算预览 diff
        ↓
TUI 展示预览
        ↓
真正 execute()
        ↓
再次读取当前文件并执行
~~~

预览和执行之间文件可能发生变化，因此最终 diff 可能不同。Pi 会以最终执行结果更新展示。

预览只是可观察性，不是审批流程，不会自动暂停等待用户确认。

## 14. EditOperations 与远程预览边界

执行层允许宿主替换：

~~~typescript
interface EditOperations {
  access(path): Promise<void>
  readFile(path): Promise<Buffer>
  writeFile(path, content): Promise<void>
}
~~~

默认使用本机文件系统；SSH Extension 可以把真正的读取和写入绑定到远程主机。模型仍然只提供 `path + edits[]`，不能通过路径动态选择后端。

但是 TUI 预览函数 `computeEditsDiff()` 直接使用本机 `fs.access()` 和 `fs.readFile()`，没有接收注入的 `EditOperations`：

~~~text
真正 execute → 可以使用远程 EditOperations
TUI 预览     → 当前实现读取本地文件系统
~~~

因此 SSH 工作区与本地文件内容不一致时，预览可能失败或展示本地版本，而实际修改发生在远程。这说明 Operations 解耦没有完全贯穿 UI 预览层。

## 15. 并发和外部修改边界

`edit` 与 `write` 共享同文件修改队列。第二个 edit 会等待第一个完成后重新读取文件，因此同一 Pi 进程中针对不同区域的并发 edit 可以依次保留下来。

但是队列不覆盖其他进程：

~~~text
Pi 读取当前文件
        ↓
外部编辑器修改文件
        ↓
Pi 将内存中的完整新内容写回
~~~

从读取到写回之间发生的外部修改可能被覆盖。`oldText` 能检测执行开始前的部分冲突，但没有保护 read-modify-write 中间窗口。

和 `write` 相同，Abort 不等于回滚。在途 `writeFile()` 可能完成副作用后，工具才报告 `Operation aborted`；修改队列会保持到在途操作结束。

## 16. 删除和插入的表达方式

删除通过空 `newText` 表达：

~~~json
{
  "oldText": "需要删除的内容",
  "newText": ""
}
~~~

空 `oldText` 被禁止，因此插入必须依附于已有唯一文本锚点：

~~~json
{
  "oldText": "const a = 1;",
  "newText": "const a = 1;\nconst b = 2;"
}
~~~

所以每次变更都必须绑定到当前存在的文本区域。

## 17. 设计取舍

### 优点

- 不依赖容易漂移的行号；
- `oldText` 提供文本级修改前置条件；
- 多个 edits 一次验证、一次写回；
- 一个 edit 失败时不会部分应用其他 edits；
- 保留 UTF-8 BOM 和统一换行风格；
- fuzzy matching 能容忍模型复制产生的 Unicode 差异；
- 提供紧凑 display diff 和标准 unified patch；
- Operations 可以替换执行后端。

### 代价和限制

- 最终仍然是完整文件写回，不是原子事务；
- 没有文件版本号或 hash；
- 外部并发修改可能丢失；
- fuzzy matching 可能规范化修改涉及行的其他字符；
- 远程 Operations 没有完全覆盖 TUI 预览路径；
- Abort 不保证撤销副作用；
- 原生路径处理和权限不构成 cwd 沙箱。

## 18. 核心设计结论

~~~text
模型声明：
“只有文件仍包含这段旧内容时，
才把它替换成新内容”
        ↓
Harness：
匹配、验证、有限容错、统一应用、写回、生成 diff
~~~

`edit` 最准确的定义是：

> 一个带文本前置条件、批量验证和可视化 diff 的 read-modify-write 工具。

## 19. 代码证据

~~~text
Schema、Prompt、参数兼容、执行和 TUI：
packages/coding-agent/src/core/tools/edit.ts

精确 / 模糊匹配、替换、换行处理和 diff：
packages/coding-agent/src/core/tools/edit-diff.ts

同文件修改队列：
packages/coding-agent/src/core/tools/file-mutation-queue.ts

基础、批量、模糊匹配、CRLF 和 BOM 测试：
packages/coding-agent/test/tools.test.ts

旧参数兼容测试：
packages/coding-agent/test/edit-tool-legacy-input.test.ts

TUI diff 预览测试：
packages/coding-agent/test/edit-tool-no-full-redraw.test.ts

并发和取消测试：
packages/coding-agent/test/file-mutation-queue.test.ts

可选 SSH 后端示例：
packages/coding-agent/examples/extensions/ssh.ts
~~~

