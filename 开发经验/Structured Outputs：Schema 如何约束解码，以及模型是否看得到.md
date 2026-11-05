---
created: 2026-09-30
tags:
  - 开发经验
  - LLM
  - Structured-Outputs
  - vLLM
  - XGrammar
  - 推理服务
---

# Structured Outputs：Schema 如何约束解码，以及模型是否看得到

> `response_format` 中的 Schema 不必直接拼进模型的输入 Token 序列。它可以作为独立的解码约束，决定每一步哪些输出 Token 合法；模型则在合法候选中决定更可能生成什么。**格式合法与内容正确是两件事。**

记录来源：2026-09-30 关于 OpenAI API、vLLM、XGrammar 与模型输入序列的讨论；写入时对照了 OpenAI 和 vLLM 的公开资料。以下是机制笔记，不代表已经对某个正在运行的服务做过请求级验证。

## 1. 最初的问题

调用 API 时可以把 JSON Schema 放在 `response_format` 或结构化输出参数中，但 Transformer 最终处理的是 Token 序列。那么 Schema 到底去了哪里？

可能的直觉是：

```text
system prompt + user query + schema + “请按此格式回答”
```

这是一种可能的提示方式，却不是实现格式约束的必要条件。需要分开看**模型收到的提示**和**采样器收到的约束**。

## 2. 解码约束怎样起作用

以 vLLM 使用 XGrammar 作为结构化输出后端为例，可以把流程简化为：

```text
messages ──> chat template / tokenizer ──> Transformer ──> logits ─┐
                                                                  ├─> 屏蔽非法 Token ─> 采样 ─> 下一个 Token
response_format / JSON Schema ──> 约束后端 ──> 当前合法 Token 集合 ─┘
```

每一步，模型根据提示词和已生成的前缀给出整个词表的分数。约束后端根据 Schema 和同一输出前缀，判断哪些 Token 仍能组成合法结果；非法 Token 的分数在采样前被屏蔽。

$$
\ell'_t(i)=
\begin{cases}
\ell_t(i), & i\in A(\text{schema},y_{<t})\\
-\infty, & i\notin A(\text{schema},y_{<t})
\end{cases}
$$

这里的 $A$ 是当前前缀下的合法 Token 集合。例子中常说“对象开头只能生成 `{`”，只是示意；实际 Token 可能包含空白、多个字符等，合法集合由具体 tokenizer 和语法状态决定。

这意味着：**强制符合结构**不要求模型先把整个 Schema 当作普通文字读进去。vLLM 的公开代码会把 `response_format` 规范化为结构化输出参数，再交给相应后端；XGrammar 路径会把语法约束生成的 bitmask 应用到 logits。后端可以配置或自动选择，不能把每一个 vLLM 部署都笼统说成必然使用 XGrammar。

## 3. 模型没看到 Schema，为什么仍会填写字段？

假设输出被约束成：

```json
{"sentiment":"positive","score":95}
```

生成到 `{"sentiment":"` 后，这段**已生成的前缀**会进入后续自回归上下文。模型此时能够看到字段名 `sentiment`，并根据用户输入给合法候选打分。如果枚举只允许 `positive` 与 `negative`，语法约束排除其他值，模型仍需在两者之间做语义选择。

可以记成：

- **约束后端**决定当前“能输出什么”。
- **模型**决定合法候选中“更想输出什么”。
- 已生成的结构会成为后续输入的一部分，但这不等于原始 Schema 的全部内容都已进入提示词。

## 4. 结构约束和语义说明要分开

Schema 的 `type`、`required`、`enum` 等可限制结果形状和候选值。字段的 `description`，例如“置信度 0 表示完全不确定，100 表示非常确定”，主要是在解释**该填什么含义**。如果这段说明只给了约束后端、没有通过任何途径进入模型上下文，单靠语法遮罩不能让模型理解这层含义。

因此，在 vLLM 的使用场景中，若需要稳定的字段语义，最好在提示词中说明输出任务、字段含义和缺失信息的处理方式；也要检查实际使用的模型、模板和服务端是否另有提示处理。vLLM 文档也建议在提示中交代 Schema 及字段如何填写。

结构正确不等于事实正确。约束可以使 `age` 是整数，却不能证明 `23` 是真实年龄。若输入不足、Schema 又要求必须填写整数，模型可能猜值。设计 Schema 时可考虑允许 `null` 或表示“未知”的枚举，并在提示中定义使用条件。

## 5. vLLM 与 OpenAI 能确认到什么程度

### vLLM

公开实现显示，Chat Completions 的 `response_format` 会被转换为 `StructuredOutputsParams`。这条结构化输出路径是解码约束通道，不能从 API 字段的存在推断出服务端必然追加了一条“请按此 Schema 回答”的系统提示。实际模型输入仍应以所用版本、Chat Template 和请求渲染结果为准。

### OpenAI

OpenAI 公开说明了两部分：训练模型更好地理解复杂 Schema，以及用动态约束解码保证合法输出。公开资料没有明确交代 Schema 在内部是否、以及怎样被序列化成模型看到的 Token。因此不能断言它一定被拼在 system/developer 消息后，也不能断言模型完全没收到 Schema 相关信息。可确认的是：可靠的格式约束并非仅靠一句提示词。

OpenAI 文档也列出例外与边界，例如拒绝回答、输出被截断，以及 JSON 值本身仍可能出错。

## 6. 不要把工具调用与 `response_format` 混为一谈

工具调用涉及模型**选择哪个工具、怎样填写参数**。不少开源模型的 Chat Template 会把工具名、描述和参数 Schema 渲染进模型输入，使模型在生成前就能读到工具定义；服务端也可以对工具参数再施加结构化解码约束。

所以可能同时存在两条路径：

```text
工具定义 ──> Chat Template ──> 模型理解工具及参数含义
参数 Schema ──> 约束后端 ──> 限制生成的参数格式
```

具体是否如此，取决于模型模板、工具解析器和服务端实现，不能把某一套工具调用流程套用到所有 `response_format` 请求。

## 7. 排查与设计时的检查顺序

1. 先确认 API 字段：`json_object`、`json_schema`、工具参数约束并不等价；也要确认当前服务支持的 JSON Schema 子集。
2. 查看请求经过服务端后生成的实际提示文本或 Token，判断 Schema 或工具定义是否进入了模型输入。
3. 查看结构化输出后端及其配置，确认语法约束是否真正作用于采样。
4. 分开测试：JSON 是否合法、是否符合 Schema、字段值在业务上是否正确。
5. 处理拒绝、截断、信息不足等情况，避免把“格式合法”误当成“结果可信”。

## 一句话记住

> Structured Outputs 可以通过“模型给概率，语法约束给可行范围”实现；Schema 是否也作为提示被模型看到，是另一个需要按具体实现核对的问题。

## 来源

- [OpenAI：Introducing Structured Outputs in the API](https://openai.com/index/introducing-structured-outputs-in-the-api/)
- [OpenAI API：Structured model outputs](https://developers.openai.com/api/docs/guides/structured-outputs)
- [vLLM：Chat Completion protocol 中的结构化输出提取](https://docs.vllm.ai/en/latest/api/vllm/entrypoints/openai/chat_completion/protocol/)
- [vLLM：`response_format` 转结构化输出参数](https://docs.vllm.ai/en/latest/api/vllm/entrypoints/generate/base/protocol/)
- [vLLM：结构化输出配置与后端选择](https://docs.vllm.ai/en/stable/api/vllm/config/structured_outputs/)
- [vLLM：结构化输出使用示例及提示词建议](https://docs.vllm.ai/en/v0.11.2/features/structured_outputs/)

注：vLLM 的代码与默认后端会随版本变化。这里记录的是公开机制和核对方法；涉及某个真实部署时，仍应看该服务的版本与请求渲染结果。
