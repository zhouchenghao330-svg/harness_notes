# Task 工具与 Subagent Runtime 笔记

## 一、先给结论

我现在对 `task` 工具的理解是：

**它本质上不是一个“待办列表工具”，而是主 agent 用来派生、管理、监督子 agent 的运行时接口。**

也就是说，当主 agent 调用一个真正的 `task` 工具时，通常不是只生成一条文本任务，而是：

- 创建一个独立的子执行单元
- 给这个子单元分配自己的线程或 session
- 让它拥有自己的上下文、状态和生命周期
- 再由主 agent 通过显式接口观察它、更新它、取消它、读取结果

所以，`task` 的本质不是 prompt 技巧，而是 **runtime 层的委派机制**。

---

## 二、为什么很多人会误解 task

很多系统表面上也有 “task” 或 “subagent”，但其实只做了下面这种事：

- 主 agent 说：“你现在扮演研究员”
- 然后在同一段上下文里继续推理
- 最后模型返回一段看起来像“子 agent 输出”的文本

这种方式可以模拟角色切换，但它不是真正的 task runtime。

因为它没有：

- 独立的 `task_id`
- 独立的 thread / session
- 独立的状态存储
- 可查询的生命周期
- 可取消、可更新、可恢复的执行控制

所以这更像 **prompt trick**，而不是实际的子 agent。

---

## 三、一个真的 task/subagent 是什么

如果从 runtime 角度定义，一个真的 `task` 往往意味着：

**主 agent 派生出一个独立运行的子单元，子单元在主 agent 之外持续存在，并通过 runtime handle 被管理。**

这个子单元通常会有：

- 自己的 `thread_id` 或 `session_id`
- 自己的消息历史
- 自己的工具调用轨迹
- 自己的中间状态
- 自己的 budget
- 自己的权限边界
- 自己的生命周期状态

所以它不是“主 agent 的一段脑内分身”，而是系统里的一个真实对象。

---

## 四、主 agent 和子 agent 的关系

我觉得这里最重要的一点是：

**子 agent 是独立执行的，但通常不是和主 agent 平权的。**

主 agent 更像 supervisor，子 agent 更像受控 worker。

常见关系是：

- 主 agent 可以创建子 agent
- 主 agent 可以查看子 agent 状态
- 主 agent 可以给子 agent 追加指令
- 主 agent 可以取消子 agent
- 主 agent 可以读取子 agent 结果

但子 agent 往往不能：

- 任意读取主 agent 的全部私有状态
- 无限制调用所有高权限工具
- 无限递归再派生更多 agent
- 脱离 supervisor 的治理独立扩张

所以准确说法不是“两个 agent 平等聊天”，而是：

**一个 supervisor 通过 task runtime 管理若干受控的独立执行单元。**

---

## 五、子 agent 的“记忆”到底是什么意思

说子 agent “有自己的记忆”通常是对的，但要区分层次。

### 1. 运行时记忆

这是最常见也最重要的部分，包括：

- 它自己的 thread history
- 它自己的 task state
- 它产生的 artifact
- 它的 event / trace / observation

这类记忆是 task 能持续运行和被主 agent 追踪的基础。

### 2. 长期记忆

这就不一定了。

有些系统会给子 agent 挂独立 memory store，让它跨任务保留经验；  
但很多系统里，子 agent 只有当前 task 生命周期内的状态，并没有真正长期存在的人格化 memory。

所以更准确的说法是：

**子 agent 一定有自己的运行时状态，但不一定有独立的长期记忆系统。**

---

## 六、主 agent 怎么和子 agent 互动

一个设计合理的 `task` 工具，一般都会提供显式生命周期接口，而不是只靠自然语言消息。

典型接口会长这样：

- `launch` / `create`
- `check` / `get`
- `list`
- `update`
- `cancel`

它们分别对应的系统行为大致是：

### `launch`

创建一个新的 task，并启动一个独立子 agent 去执行。

### `check`

查看该 task 当前状态，比如：

- 是否还在运行
- 已经执行到什么阶段
- 有没有中间结果
- 有没有报错

### `list`

列出当前活跃或历史任务，便于 supervisor 统一治理。

### `update`

给同一个 task/thread 追加新要求。  
它不应该被理解成“往 prompt 里塞一句话”，而更像是：

**在同一个 task 上开启一次新的 run，让子 agent 基于既有状态继续工作。**

### `cancel`

中止任务，停止继续消耗资源。

---

## 七、为什么 task 不能只靠消息历史存在

这是我觉得 task runtime 设计里最关键的地方之一。

如果 task 的存在只靠主 agent 在对话里记一句：

> “我刚才派了一个子 agent 去查资料”

那么一旦发生：

- compaction
- summarization
- context trimming
- 长链路推理后的遗忘

主 agent 就可能丢失对子任务的稳定引用。

所以一个真正可用的 task 系统必须把状态外置到 runtime channel，而不是只放在 message history 里。

这通常意味着至少要有：

- `task_id`
- `thread_id`
- `run_id`
- `status`
- `result`
- `error`
- `events`
- `artifacts`

也就是说：

**主 agent 和子 agent 的关系，应该靠 runtime handle 维持，而不是靠叙事记忆维持。**

---

## 八、一个好的 task 工具最少应该具备什么

如果按工程标准看，一个最小但靠谱的 task 工具至少要有这些性质：

### 1. 身份稳定

必须有明确的 `task_id`，不能靠“刚才那个任务”这种自然语言引用。

### 2. 状态外置

状态不能只活在上下文里，要有稳定的 runtime state store。

### 3. 生命周期完整

至少支持：

- 创建
- 查询
- 更新
- 取消

### 4. 可观测

不能只有最终答案，最好还能看到：

- 事件流
- 日志
- 中间结果
- 工具调用轨迹

### 5. 预算与权限控制

task 最好自带：

- token budget
- time budget
- tool whitelist
- recursion depth limit
- sandbox / workdir

### 6. 父子关系明确

谁创建了谁，谁能看谁，谁能取消谁，要可追踪。

### 7. 失败可恢复

失败后最好支持：

- retry
- resume
- partial result recovery

---

## 九、怎么区分真假 subagent

我现在会用一个很直接的判断标准：

### 假的 subagent

- 只是换个角色 prompt
- 没有独立 thread
- 没有独立状态
- 没有生命周期管理
- 没有可观测性
- 所有东西仍然挤在主 agent 的上下文里

这种更像“多角色推理模板”。

### 真的 subagent

- 有独立 task identity
- 有独立 thread / run
- 有独立 runtime state
- 有 supervisor 可调用的管理接口
- 有明确的权限和预算边界
- 可以在主 agent 之外持续运行

这种才更接近真正的 agent runtime。

---

## 十、一句话总结

我现在最认可的定义是：

**`task` 工具 = 把“委派执行”从消息层提升到运行时层。**

它的价值不在于“让模型假装分身”，而在于：

- 让主 agent 能真正派生独立子 agent
- 让子 agent 成为可管理的运行时对象
- 让监督、追踪、预算、权限、取消、恢复都变成显式系统能力

所以，task/subagent 的关键不在“像不像多智能体”，而在：

**它有没有把子任务变成一个真实存在、可被治理的 runtime entity。**
