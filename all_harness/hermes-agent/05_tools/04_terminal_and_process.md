# Hermes Core Tool 04：terminal / process

> 状态：待共同阅读。

## 1. 研究定位

`terminal` 对应 Pi 的 `bash`，负责创建命令执行；`process` 负责继续管理已经启动的长任务或交互进程。

~~~text
terminal(command)
      ↓
选择 local / ssh / docker / sandbox 等环境
      ↓
启动 PTY / 子进程
      ↓
短命令结束 ───────────→ Tool Result
      │
      └── 长任务 / 后台任务
                ↓
           process(action)
      poll / write / terminate / inspect
~~~

## 2. 共同阅读问题

- 模型侧如何选择命令、cwd、timeout 和后台模式？
- Shell 与执行后端由模型选择，还是由 Session 预先绑定？
- local、Docker、SSH、Modal 等环境如何统一抽象？
- PTY、普通 subprocess 和 detached process 如何区分？
- stdout / stderr 是否分离，输出是否流式返回？
- 大输出保留头部、尾部还是完整外溢文件？
- `process` 如何通过 task/process id 找回运行状态？
- 命令失败、超时、取消分别怎样表示？
- 终止的是外层 Shell，还是整个进程树？
- 危险命令怎样识别并请求 Approval？
- 多条命令和多个 Tool Call 如何并发，资源冲突如何处理？
- 与 Pi `bash` 的单次 Shell 和 OutputAccumulator 相比如何？

## 3. 主要代码入口

~~~text
tools/terminal_tool.py
tools/process_registry.py
tools/environments/
tools/approval.py
tools/interrupt.py
tools/tool_backend_helpers.py
~~~
