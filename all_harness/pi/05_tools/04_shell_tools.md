# Pi Concrete Tool 04：Bash

> 状态：Bash 已共同阅读并确认；PowerShell 待共同阅读。

## 1. 工具定位

`read`、`write`、`edit` 是能力边界明确的专用工具，`bash` 则是把一整段命令交给 Shell 解释执行的通用工具。

~~~text
read / write / edit
  → Harness 理解操作语义

bash
  → Harness 不解析 ls、git、npm、rm 等命令的业务语义
  → 只负责启动 Shell、管理进程和处理输出
~~~

因此 Bash Tool 的模型接口很小，但实际权限和表达能力很大。

## 2. 模型可见接口

~~~typescript
bash({
  command: string,
  timeout?: number
})
~~~

- `command`：交给 Shell 执行的完整命令文本；
- `timeout`：可选超时秒数；
- 不提供 `timeout` 时没有默认执行超时。

定义位置：

~~~text
packages/coding-agent/src/core/tools/bash.ts:42
~~~

虽然只有两个参数，`command` 本身可以包含一小段 Shell 程序：

~~~bash
cd packages/coding-agent && npm run check
git status --short | head -20
git status > status.txt
~~~

其中：

- `;`：按顺序执行，不要求前一条成功；
- `&&`：前一条成功后才执行下一条；
- `|`：把前一条命令的输出交给下一条；
- `>`：把输出写入文件。

`ls`、`git`、`npm`、`rm` 等通常是 Shell 启动的其他程序；`cd`、变量展开、条件判断和循环等属于 Shell 自身的能力。

## 3. 一次 Tool Call 的执行链

~~~text
LLM 生成 { command, timeout? }
        ↓
拼接 commandPrefix
        ↓
构造 command / cwd / env
        ↓
可选 spawnHook 重写执行上下文
        ↓
BashOperations.exec()
        ↓
启动 Shell 子进程
        ↓
stdout ─┐
stderr ─┴→ OutputAccumulator
        ↓
流式更新 + 最终 Tool Result
        ↓
写回 Agent Context
~~~

在 macOS/Linux 上，Pi 优先使用 `/bin/bash -c <command>`；找不到 Bash 时回退到 `sh`。Windows 会寻找 Git Bash 或 PATH 中的 Bash。PowerShell 是另一个独立工具，不是 Bash 在 Windows 上的默认替代名称。

Shell 选择逻辑：

~~~text
packages/coding-agent/src/utils/shell.ts
~~~

## 4. cwd 和 Shell 状态

每次 Bash Tool Call 都会启动一个新的 Shell 进程。一次调用里的 `cd` 只影响这一次 Shell：

~~~text
bash("cd packages")
        ↓
Shell A 改变目录后结束

bash("pwd")
        ↓
Shell B 仍从工具预先绑定的 cwd 启动
~~~

因此需要在同一次调用中表达目录切换：

~~~bash
cd packages/coding-agent && npm run check
~~~

Shell 自己的临时状态通常不会跨调用保留，但外部副作用会保留，例如：

- 已创建、修改或删除的文件；
- 已发送的网络请求；
- 已提交的数据库修改；
- 成功脱离进程组的后台服务。

## 5. 成功、失败和退出码

Shell 进程结束后会返回退出码：

~~~text
exit code = 0
  → Tool Result 成功

exit code != 0
  → 保留命令输出
  → 抛出 Command exited with code N
  → Agent Loop 转换成 isError: true 的 Tool Result
~~~

成功但没有输出时，模型收到：

~~~text
(no output)
~~~

Pi 把 `stdout` 和 `stderr` 接入同一个 `onData` 回调。因此模型看到的是合并后的输出，不能可靠地区分某一段来自标准输出还是错误输出。

## 6. 超时、取消和进程树

模型可以显式指定：

~~~json
{
  "command": "npm run check",
  "timeout": 60
}
~~~

超时或收到 AbortSignal 时，Pi 不只终止外层 Shell，而是尝试终止整个进程树：

~~~text
bash
  └── npm
       └── node
            └── test worker
~~~

- macOS/Linux：Shell 以独立进程组启动，取消时向进程组发送 `SIGKILL`；
- Windows：使用 `taskkill /F /T`；
- 如果进程组终止失败，退回到只终止直接子进程。

Pi 还跟踪 detached 子进程 PID，以便主进程退出时清理。

取消只停止仍在运行的进程，不会回滚已经发生的副作用：

~~~bash
echo hello > result.txt
sleep 100
~~~

如果在 `sleep` 阶段取消，`result.txt` 仍然存在。

相关代码：

~~~text
packages/coding-agent/src/core/tools/bash.ts:84
packages/coding-agent/src/utils/shell.ts
packages/coding-agent/src/utils/child-process.ts
~~~

## 7. 流式输出

命令运行期间，`stdout` 和 `stderr` 会持续进入 `OutputAccumulator`。Pi 不会等命令彻底结束后才展示输出。

为了避免高频输出造成大量事件和 UI 重绘，工具把更新频率限制为最多约每 100ms 一次：

~~~text
进程连续输出大量 chunk
        ↓
OutputAccumulator 持续累计
        ↓
100ms 节流合并
        ↓
发送 tool_execution_update
~~~

命令结束后会立即刷新最后一次尚未发出的内容。

## 8. 输出截断与完整临时文件

模型侧结果默认最多保留：

~~~text
最后 2000 行
或者
最后 50KB
以先触发的限制为准
~~~

Bash 使用 tail truncation，优先保留末尾，因为编译错误、测试失败汇总、异常栈和退出原因通常出现在最后。

只有输出超过上述任一限制时，Pi 才创建临时文件：

~~~text
输出未超过限制
  → 全部直接返回
  → 不创建临时文件

输出超过 2000 行或 50KB
  → 完整原始输出保存到 /tmp/pi-bash-<random>.log
  → Context 只接收尾部
  → Tool Result 附带完整文件路径
~~~

模型看到的提示类似：

~~~text
[Showing lines 8001-10000 of 10000.
 Full output: /tmp/pi-bash-a81f.log]
~~~

这个路径只是按需二次调查的入口。Harness 不会自动重新读取或核对完整输出。模型如果认为尾部信息不足，可以主动执行：

~~~bash
grep -n "ERROR" /tmp/pi-bash-a81f.log
~~~

或者通过 `read(path, offset, limit)` 分页读取。

超时、取消或失败前已经产生大量输出时，也会保留完整临时文件，并把路径附加到错误消息。当前 `OutputAccumulator` 只负责创建和关闭写入流，没有主动删除临时文件。

相关代码：

~~~text
packages/coding-agent/src/core/tools/bash.ts:355
packages/coding-agent/src/core/tools/output-accumulator.ts
~~~

## 9. 为什么 Bash 输出需要保留完整副本

`read` 的内容通常可以通过同一路径和 offset 再次读取，但 Bash 输出可能无法复现：

~~~text
一次性的编译错误
实时日志
测试运行结果
远程命令输出
随时间变化的系统状态
~~~

因此 Pi 采用：

~~~text
Context 中只保留有价值的尾部
        +
临时文件中保留完整输出
~~~

这是 Context 成本与可追溯性之间的折中。

## 10. 宿主侧配置和扩展点

模型只看到 `command` 和 `timeout`。宿主创建 Bash 工具时还可以配置：

~~~typescript
interface BashToolOptions {
  operations?: BashOperations
  commandPrefix?: string
  shellPath?: string
  exposeSessionEnvironment?: boolean
  spawnHook?: (context) => context
}
~~~

### commandPrefix

把初始化命令添加到每次模型命令之前：

~~~text
commandPrefix
      + 换行
模型 command
~~~

它适合设置 alias、环境变量或固定的 Shell 初始化逻辑。

### spawnHook

在启动进程前修改：

~~~text
command
cwd
env
~~~

它可以统一增加环境变量、改写工作目录或包裹命令。

### PI_* 环境变量

默认情况下，Pi 会向模型 Bash Tool 注入当前会话信息：

~~~text
PI_SESSION_ID
PI_SESSION_FILE
PI_PROVIDER
PI_MODEL
PI_REASONING_LEVEL
~~~

宿主可以通过 `exposeSessionEnvironment: false` 禁用。Pi 会先删除继承环境中旧的同名变量，再根据当前 Session 写入，避免子进程读到过期会话信息。

## 11. BashOperations：执行语义与后端解耦

~~~typescript
interface BashOperations {
  exec(command, cwd, {
    onData,
    signal,
    timeout,
    env
  }): Promise<{ exitCode: number | null }>
}
~~~

默认实现启动本机 Shell，但宿主或 Extension 可以提前替换为 SSH、容器或 VM 执行器：

~~~text
模型 bash(command, timeout)
          ↓
同一个 Bash Tool 逻辑
          ↓
预先绑定的 BashOperations
     ┌────┼─────┐
     ↓    ↓     ↓
 Local  SSH   VM/Container
~~~

这仍然是宿主侧依赖注入，不是模型通过参数动态选择执行后端。一个普通 Bash Tool 实例通常预先绑定一个执行环境。

当然，模型也可以在本地 Bash 中直接生成：

~~~bash
ssh user@server 'cat /path/file'
~~~

这是 Shell 命令本身启动 SSH，和宿主替换 `BashOperations` 是两种不同机制。

## 12. 安全边界

Pi 的 Bash Tool 本身没有命令 allowlist、denylist 或 `rm -rf` 特殊检测：

~~~text
模型生成 rm -rf ...
        ↓
Bash Tool 不理解它是删除操作
        ↓
Shell 启动 rm
        ↓
权限边界等于 Pi 进程的操作系统权限
~~~

因此 Bash 可以：

- 读取、修改和删除文件；
- 访问继承到进程中的环境变量；
- 发起网络请求；
- 安装依赖或运行项目脚本；
- 启动其他进程。

Pi 明确不把进程内检查包装成安全沙箱。真正的安全隔离需要来自操作系统、容器、VM、远程沙箱、挂载策略和凭据限制。

文档依据：

~~~text
packages/coding-agent/docs/security.md
~~~

## 13. 模型 Bash Tool 与用户 `!command`

Pi 还有一条用户直接输入 `!command` 的执行路径。两者可以复用同一个 `BashOperations` 后端，但不是同一个上层调用：

~~~text
模型 Tool Call
  → core/tools/bash.ts
  → OutputAccumulator
  → ToolResultMessage

用户 !command
  → AgentSession.executeBash()
  → core/bash-executor.ts
  → BashExecutionMessage / Session History
~~~

因此研究模型工具行为时，应以 `core/tools/bash.ts` 为主；`bash-executor.ts` 主要描述用户或模式层直接执行 Shell 命令的路径，不能把两条路径的全部细节混为一谈。

## 14. 设计总结

~~~text
极小模型接口
  command + timeout
        ↓
Shell 提供强大表达能力
        ↓
Pi 负责进程生命周期和输出治理
        ↓
Operations / Hook 支持宿主重绑定执行环境
~~~

优点：

- 接口简单，却能调用整个开发工具链；
- 支持流式输出、取消、超时和进程树终止；
- 大输出不会直接撑爆 Context；
- 执行后端可以由宿主替换。

代价和限制：

- 权限极高，内置工具本身没有安全隔离；
- Shell 命令字符串比结构化专用工具更难验证；
- stdout 和 stderr 合并；
- 取消不会回滚副作用；
- Shell 状态不跨 Tool Call 持久化；
- 完整临时输出不会被 Harness 自动复查。
