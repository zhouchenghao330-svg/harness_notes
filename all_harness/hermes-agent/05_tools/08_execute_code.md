# Hermes Core Tool 08：execute_code

> 状态：待共同阅读。

## 1. 研究定位

`execute_code` 是 Hermes 的 Meta Tool：模型生成 Python，不是为了获得一个任意宿主 Python Shell，而是为了在受控执行环境里编排一组允许的工具。

~~~text
模型生成 Python
      ↓
受限 Code Runtime
      ↓
程序化调用允许的工具
  ├── Tool A
  ├── Tool B
  └── Tool C
      ↓
一次综合结果返回 Agent Loop
~~~

## 2. 共同阅读问题

- 模型侧 Schema 接收代码、输入和 timeout 的方式是什么？
- Python 在进程内、子进程、Kernel 还是 Sandbox 中执行？
- 哪些工具允许被代码调用，allowlist 如何从当前 Session Tool Snapshot 派生？
- 是否可能借此重新获得被 Toolset 禁用的能力？
- 工具调用怎样继承 session、task、approval 和 cancellation？
- 内部多次工具调用怎样计入预算和 trace？
- stdout、返回值、异常和工具结果怎样合并？
- 长输出如何截断或外溢？
- 取消代码时能否终止内部正在运行的工具？
- 为什么这种 Meta Tool 比多轮 Tool Call 更便宜，又增加了哪些安全风险？

## 3. 主要代码入口

~~~text
tools/code_execution_tool.py
tools/code_kernel.py
model_tools.py
tools/registry.py
~~~
