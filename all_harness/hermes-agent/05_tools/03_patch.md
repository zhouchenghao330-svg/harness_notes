# Hermes Core Tool 03：patch / edit

> 状态：已共同阅读。
>
> 一句话定位：`patch` 是一种 **带旧内容前置条件的局部修改协议**——只有目标内容还能在当前文件里找到，修改才会继续。

## 1. 先记住三点

~~~text
1. patch 比 write_file 安全：
   它不会默认覆盖整个文件，而是先寻找旧内容。

2. Hermes 会容忍空格、缩进和 Unicode 差异，
   但不会理解 Python 语法并替模型修正错误的内部缩进。

3. 多文件 Patch 会先验证全部操作；
   但真正写入时仍可能部分成功，没有自动回滚。
~~~

## 2. 为什么它比 `write_file` 安全

`write_file` 表达的是：

~~~text
把整个文件替换成 X
~~~

`patch` 表达的是：

~~~text
找到旧内容 A
      ↓
确认 A 仍然存在
      ↓
只把 A 换成 B
~~~

这里的 `old_string` 实际是一项前置条件：

> 文件局部状态仍然符合模型看到的旧版本，才执行修改。

如果别人已经改掉这段内容，Patch 通常会失败，而不是像整文件写入那样直接覆盖。

## 3. 两种 Patch 模式

### 3.1 Replace 模式：单文件局部替换

~~~json
{
  "mode": "replace",
  "path": "app.py",
  "old_string": "return 1",
  "new_string": "return 2"
}
~~~

可以设置 `replace_all=true`，但默认要求只找到一个位置。

### 3.2 V4A 模式：多文件结构化修改

~~~text
*** Begin Patch
*** Update File: app.py
-return 1
+return 2
*** Add File: config.py
+DEBUG = True
*** End Patch
~~~

V4A 实际支持：

~~~text
Update File
Add File
Delete File
Move File
~~~

所以它不只是 edit，也能创建、删除和移动文件。

## 4. Replace 模式怎样执行

~~~text
读取当前完整文件
       ↓
查找 old_string
       ↓
确认匹配位置唯一
       ↓
生成替换后的完整内容
       ↓
通过 write_file 的原子写入流程落盘
       ↓
重新读取验证
       ↓
返回 unified diff + lint / LSP 结果
~~~

`old_string` 不能为空、不能只有空格，也不能和 `new_string` 完全相同。

## 5. 模糊匹配怎样做

Hermes 不是直接计算一个相似度，然后随便选择代码块。它按“从严格到宽松”的顺序逐层尝试：

~~~text
精确匹配
   ↓ 找不到
容忍行首行尾空格
   ↓
容忍空格数量和缩进差异
   ↓
处理转义与 Unicode 差异
   ↓
最后才使用代码块 / 上下文近似匹配
~~~

对应的九种内部策略是：

~~~text
exact
line_trimmed
whitespace_normalized
indentation_flexible
escape_normalized
trimmed_boundary
unicode_normalized
block_anchor
context_aware
~~~

它会停在第一个找到匹配的策略，不会把九种结果混合评分。

## 6. 怎样避免模糊匹配改错位置

最重要的保护是唯一性：

~~~text
找到 1 个位置
  → 可以修改

找到多个位置
  → 拒绝修改
  → 要求模型提供更多上下文
~~~

`replace_all=true` 也有限制：

~~~text
多个精确匹配
  → 可以全部替换

多个近似匹配
  → 拒绝全部替换
~~~

Hermes 还会检查异常的引号、反斜杠和转义，避免 Tool Call 序列化差异被直接写进源码。

不过仍然存在一个边界：

> 文件里唯一的“相似代码块”，不一定就是模型真正想修改的位置。

因此 `old_string` 仍应包含足够上下文，而不是只给一个常见的 `return None`。

## 7. 缩进到底会不会自动修

答案是：**只修整块代码的基础缩进，不修代码内部的语法层级。**

### 7.1 整块缩进偏移：会修

文件真实内容：

~~~python
class Calculator:
    def add(self):
        result = 1
        return result
~~~

模型发送的是零缩进版本：

~~~python
result = 1
return result
~~~

Hermes 通过模糊匹配发现目标区域实际位于 8 个空格处，会把整个 `new_string` 平移到同一深度，同时保留模型各行之间的相对缩进。

### 7.2 内部层级写错：不会修

模型提交：

~~~python
if ready:
print("ready")
return True
~~~

Hermes 不理解 `print` 应位于 `if` 内部。它没有解析 Python AST，只会整体移动代码块。

Python Lint 可能在写入后发现 `IndentationError`，但文件已经落盘，不会自动回滚。

### 7.3 精确匹配成功：完全不自动调整缩进

自动 re-indent 只发生在非精确的模糊匹配中。

如果 `old_string` 精确匹配，Hermes 会直接写入 `new_string`；即使 `new_string` 缩进错误，也不会替模型修正。

可以压成：

~~~text
Hermes 能修：整块代码放错深度
Hermes 不能修：代码块内部缩进逻辑错误
~~~

## 8. 重复 Patch 为什么不会一直失败

假设第一次已经完成：

~~~text
return 1 → return 2
~~~

模型因为 Nudge、重试或 Session 恢复，又发送相同 Patch。

这时：

~~~text
old_string 已经消失
new_string 已经存在
~~~

Hermes 会返回：

~~~text
success = true
no_change = true
编辑似乎已经应用，不要再次重试
~~~

这让 Patch 具有一定的重复执行安全性，避免恢复机制把同一修改反复执行。

## 9. 多文件 V4A 的“两阶段”设计

### 第一阶段：全部验证，不写文件

~~~text
读取所有目标文件
      ↓
在内存中模拟所有 Hunk
      ↓
检查删除和移动目标
      ↓
任意操作验证失败
      ↓
整个 Patch 拒绝，一个文件都不改
~~~

后面的 Hunk 会基于前面 Hunk 的模拟结果继续验证，因此同一文件里的多个修改可以依赖前面的修改。

### 第二阶段：按顺序真正执行

~~~text
修改 A 成功
修改 B 成功
修改 C 失败
~~~

Hermes 不会自动恢复 A、B，而是返回：

~~~text
Apply phase failed
state may be inconsistent
run git diff to assess
~~~

所以：

~~~text
验证阶段
  → 全有或全无

真正写入阶段
  → 可能部分成功
~~~

单个 Update 文件仍使用 `write_file` 的临时文件原子替换，但多个文件合起来不是一个事务。

## 10. 并发和 stale 状态

Hermes 会：

~~~text
提前收集所有目标路径
按固定顺序获取 per-path lock
避免两个 Patch 交叉修改同一路径
~~~

固定排序获取多个锁，是为了避免两个多文件 Patch 互相等待造成死锁。

但是和 `write_file` 相同：如果发现文件在读取后被别人修改，当前主要是附加 stale warning，仍然继续尝试 Patch。

Patch 比整文件写入稍安全，因为旧内容如果已经变化，匹配通常会失败；但模糊匹配仍可能找到一个近似位置。

## 11. 最终返回什么

成功结果主要包含：

~~~text
success
unified diff
files_modified / created / deleted
lint
lsp_diagnostics
resolved_path
~~~

找不到旧内容时，Hermes 会提示重新读取或搜索文件。连续多次失败后，提示会升级，要求模型停止猜测并重新取得当前内容。

## 12. 最终记忆模型

~~~text
当前文件
   ↓
old_string 作为局部前置条件
   ↓
从精确到宽松逐层匹配
   ↓
确认匹配位置唯一
   ↓
保留相对缩进并生成新内容
   ↓
通过 write_file 原子写入单个文件
   ↓
返回 diff 与验证结果
~~~

最值得记住的是：

> `patch` 不是普通字符串替换，而是“带前置条件、容错定位和结果证据的局部 Mutation Protocol”。

主要代码入口：

~~~text
tools/file_tools.py
  PATCH_SCHEMA
  patch_tool()

tools/fuzzy_match.py
  九级匹配
  唯一性检查
  re-indent
  already-applied detection

tools/patch_parser.py
  V4A 解析
  validate-then-apply

tools/file_operations.py
  patch_replace()
  patch_v4a()

tools/file_state.py
agent/tool_dispatch_helpers.py
~~~
