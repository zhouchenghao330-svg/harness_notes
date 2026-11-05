---
created: 2026-09-20
tags:
  - 开发经验
  - Agent
  - System-Prompt
  - 上下文工程
  - Prompt-Cache
---

# Agent 中途更新 System Prompt：消息协议、历史重放与缓存代价

> Agent 可以在对话记录里保存一条中途的 system 变更，但这不意味着模型服务端接受原位发送的 `role: system`。每次模型请求前应按目标模型的能力转换记录：支持则原位发送；不支持则合成开头的 system，并接受由此前缀变化带来的缓存代价。

记录来源：2026-09-20 对 AI Human Runtime、pi 0.86.0 源码的核对，以及在 AI Human 开发 Docker 容器中对当前 `qwen38` 服务的真实送模实验。

## 1. 先区分三层

```text
Agent 对话记录
  可以记录：原始 system、工具调用、工具回执、后续 system 变更
        ↓ 每次请求前按模型能力转换
实际 API 请求
  可能保留中途 system，也可能合并成开头 system
        ↓ 模型服务端校验、转换消息并套用 chat template
模型输入
```

“对话记录里有一条中途 system”“LangChain 能构造 `SystemMessage`”“模型服务端接受这种顺序”是三个不同的判断。只看内存中的消息列表，不能推断最终请求会被接受。

这里的“中途”指同一个用户请求的工具循环中，模型先调用工具，Agent 观察工具结果后，下一次模型请求前追加规则；不是修改一条已经开始生成的请求。

## 2. 当前 AI Human 的实测结果

实验在开发 API 容器中读取当前环境配置，使用 `qwen38`、`vllm_qwen`，通过固定的实验工具 `lookup_probe_token` 返回 `OBSERVATION_42`。没有提交正式 run、调用业务工具或写数据库。

| 送模路径 | 第二次请求的消息顺序 | 结果 |
|---|---|---|
| 直接 LangChain，工具回执后追加 `SystemMessage` | system → user → assistant 工具调用 → tool 回执 → system | 服务端返回 400：`System message must be at the beginning.` |
| AI Human `agent_loop`，第二次请求前动态追加 `SystemMessage` | 同上 | 同样返回 400 |
| 两条路径的对照请求：把新规则合入开头 system | system → user → assistant 工具调用 → tool 回执 | 请求成功；回答同时包含工具结果与新规则要求的标记 |
| 绕过 LangChain，直接用 OpenAI SDK 发送中途 `role: system` | system → user → assistant → system → user | 同样返回 400；把规则放到开头 system 后，请求被接受 |

因此，当前实验可以确定：**LangChain 没有在本次路径中拒绝或自动前置这条消息；当前模型服务端拒绝了原位的中途 system。** 原生 SDK 的相同报错把 LangChain 从这次 400 的直接来源中排除了。

这并不证明所有 Qwen 模型或所有部署都如此，也不能仅凭客户端收到的 400 精确定位到远端 vLLM 的哪段校验、哪份实际加载的 chat template，更不能归因于模型权重。`qwen38` 是当前服务标识，不直接等同于官方模型版本名。

## 3. LangChain 在这里做了什么

AI Human 使用的本地 `langchain_openai` 适配器默认把 `SystemMessage` 转成 `role: "system"`，再按输入消息的顺序逐条构造 Chat Completions 请求。它不会自动判断目标模型是否支持中途 system，也不会自动把这条消息改成 user 或合并到开头。

所以这次链路是：

```text
工具回执后追加 SystemMessage
    → LangChain 原位映射为 role: system
    → 当前服务端返回 400
```

AI Human 还有另一项独立限制：持久化历史进入后续普通聊天时，ContextManager 的历史投影和 LLMClient 的消息构造没有把历史中的 system 作为可重放的变更保留下来。一次请求前临时插入、同一 run 的下一次请求、跨 turn 持久重放，需要分别检查；不能把“某一步可以 append”当成三者都已支持。

## 4. pi 如何兼容不支持中途 system 的模型

pi 0.86.0 把原始 system 和后续变更保存在 transcript 中。后续 system 可以追加指令、按名称更新或移除提示词章节，也可以记录工具变化。它在**下一次模型请求前**检查模型能力：

- `supportsMidConvoSystemMessages` 为真：后续 system 保留在记录中的位置，适配器发送对应的 system 或 developer 消息。
- 为假或未声明：重放全部 system 变更，合成一条新的开头 system；请求中的后续 system 移除，非 system 的对话历史继续发送。

OpenAI Completions 兼容配置默认是 `false`。pi 没有把中途 system 偷换成 user；即使走原位发送分支，适配器使用的也是 system 或 developer 角色。对于当前实测会返回 400 的服务端，pi 式做法应走“合并到开头”的分支。

需要注意：pi 保存的是**变更记录**，发给模型的是**按目标能力生成的本次投影**。同一条 transcript 可以因模型能力不同而生成不同请求。

## 5. 合并到开头的缓存代价

假设上一次请求是：

```text
[原 system][用户消息][助手工具调用][工具回执]
```

新增 system 变更后，若服务端不支持原位发送，下一次实际请求变为：

```text
[原 system + 新规则][用户消息][助手工具调用][工具回执]
```

虽然旧对话正文相同，**请求前缀已经从开头的 system 处改变**。此前整段历史对应的 prefix/KV cache 通常无法继续作为完全相同的前缀命中；服务端能否复用变化前的一小段，取决于分词、消息模板、缓存粒度和实现，不能保证。`prompt_cache_key` 或会话标识可以参与缓存路由，但不会把内容不同的前缀变成相同内容。

如果目标服务端真正支持原位发送中途 system，新增规则放在旧历史之后，旧请求前缀便有机会保持不变并被复用。当前 `qwen38` 接口拒绝这种消息顺序，因此这条路在当前部署不可用。

## 6. 可复用的排查方法

遇到“中途 system 不生效”或 400，不要只看 Agent 内存里的消息：

1. 确认变更是在当前请求生成前加入，还是错误地试图影响正在生成的请求。
2. 分别记录持久化 transcript、发送前的消息投影、最终 API 请求中的角色顺序。
3. 用原生 SDK 向同一服务端发送最小对照请求，区分适配器转换问题与服务端拒绝。
4. 同时测试原位 system 和合并到开头 system；记录响应状态、错误原文、是否实际执行工具。
5. 检查上下文压缩、预算整理和历史重放是否再次改写或丢弃临时消息。
6. 如需优化缓存，比较变更前后的实际 token 前缀及服务端缓存用量，不能只凭会话 ID 推断命中。

这次只验证了当前服务端的 Chat Completions 路径；没有对其他 Qwen 部署、Responses 路径的真实服务端行为或实际缓存命中率作通用结论。

## 一句话记住

> 把 system 变更保存在 Agent 记录中，再按模型能力决定下一次请求如何发送。当前服务端不接受中途 system，就只能合并到开头；请求能成功，但旧历史的完整缓存前缀随之改变。

## 来源

- [AI Human 真实送模复现实验](/Users/zhouchenghao/python_file/wj/ai-human/labs/mid_system_probe.py)
- [AI Human 工具循环的请求准备](/Users/zhouchenghao/python_file/wj/ai-human/app/agent_runtime/agent_loop.py)
- [AI Human 历史投影](/Users/zhouchenghao/python_file/wj/ai-human/app/agent_runtime/context_manager.py)
- [AI Human 模型消息构造](/Users/zhouchenghao/python_file/wj/ai-human/app/agent_runtime/core/llm.py)
- [LangChain OpenAI 适配器的角色映射与请求构造](/Users/zhouchenghao/python_file/wj/ai-human/langchain_openai/chat_models/base.py)
- [pi transcript 的变更重放和能力分流](/Users/zhouchenghao/python_file/pi/packages/ai/src/utils/transcript.ts)
- [pi system 消息结构](/Users/zhouchenghao/python_file/pi/packages/ai/src/types.ts)
- [pi Agent 循环处理待发送消息](/Users/zhouchenghao/python_file/pi/packages/agent/src/agent-loop.ts)
- [pi OpenAI Completions 兼容能力与消息转换](/Users/zhouchenghao/python_file/pi/packages/ai/src/api/openai-completions.ts)
- [pi OpenAI Responses 的消息转换](/Users/zhouchenghao/python_file/pi/packages/ai/src/api/openai-responses-shared.ts)

注：以上源码链接指向本机检出的工作区；文件移动或服务配置变化后，应重新核对。原生 SDK 对照实验在容器中执行，未保存为脚本。
