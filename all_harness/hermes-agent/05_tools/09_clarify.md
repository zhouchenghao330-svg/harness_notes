# Hermes Core Tool 09：clarify

> 状态：待共同阅读。

## 1. 研究定位

`clarify` 不只是返回一段问题文本，而是把 Agent 的自动执行切换成人工输入边界：

~~~text
Agent 信息不足
      ↓
clarify(question / choices)
      ↓
Runtime 向当前 Surface 发出交互请求
      ↓
等待用户输入或超时
      ↓
回答成为 Tool Result
      ↓
同一 Agent Loop 继续
~~~

## 2. 共同阅读问题

- Schema 如何表达开放问题和选项问题？
- CLI、Gateway、TUI、Desktop 分别怎样展示并收集回答？
- 等待期间 Agent Run、锁、deadline 和取消处于什么状态？
- 回答如何关联正确的 session、turn 和 tool_call_id？
- 用户断线、进程重启或超时后怎样恢复？
- 当前 Surface 不支持交互时怎样降级？
- Clarify 是阻塞当前执行线程，还是挂起后由事件恢复？
- 回答怎样写入 Tool Result 和持久化 transcript？
- 多个并发 Clarify 是否允许？
- 与普通 assistant 问句相比，为什么需要专用工具？

## 3. 主要代码入口

~~~text
tools/clarify_tool.py
tools/clarify_gateway.py
gateway/
tui_gateway/
~~~
