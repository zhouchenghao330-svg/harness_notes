---
created: 2026-09-21
tags:
  - 开发经验
  - LLM
  - Reasoning
  - Chat-Template
  - vLLM
  - 上下文工程
---

# 推理模型中文思考控制：Chat Template、预填与参数映射

> 最终正文是中文，不代表 reasoning 一定是中文。要稳定控制推理语言，需要同时核对应用层语言合同、请求参数、服务端 Chat Template 和真正的生成起点。排查时不要只搜索业务 Prompt，要查看模型服务最终渲染出的 token 输入。

记录来源：2026-09-21 对 AI Human Runtime 与 Qwen38 vLLM 服务的真实排查、模板修改和流式生成验证。

## 1. 问题现象

同一个推理模型出现了不稳定行为：

- 最终回答始终是中文。
- 简单问候的思考过程是中文。
- 换成工具能力等问题后，思考过程变成英文。

业务 Prompt 已经基本是中文，所以直觉上容易把问题归因于模型本身，或者继续在 System Prompt 中重复“请使用中文”。但真实链路里，模型看到的内容并不只来自业务代码：

```text
业务角色 Prompt
    + 应用层语言合同
    + Provider 请求参数
    + 服务端 Chat Template 注入内容
    + assistant 生成起点
    → 最终 token 输入
```

只检查第一层，很容易漏掉服务端在请求到达后追加的英文指令。

## 2. 正文语言和思考语言是两个约束

很多语言合同只写：

```text
最终回复使用中文。
```

这对 reasoning 模型并不完整。模型可能用英文推理，再把最终结论写成中文。如果产品会把 reasoning 展示给用户，就必须把它纳入显式输出合同：

```text
思考过程和最终回答都必须使用中文。
```

机器可读协议应排除在自然语言翻译范围之外：

- JSON 键
- 枚举值
- 工具名称
- 参数名和字段名
- ID
- XML/JSON 工具调用格式

否则为解决语言问题而翻译工具协议，可能破坏 parser 和工具执行。

## 3. 为什么只改 System Prompt 仍可能不稳定

System Prompt 中的中文要求通常距离实际生成位置较远。服务端还可能在它之前或之后加入英文模板，例如：

```text
Reasoning effort is set to xhigh. Please think carefully...
```

而最终生成边界可能只是：

```text
<|im_start|>assistant
<think>
```

模型开始生成时没有明确的中文续写形态，不同问题就可能激活不同语言习惯。因此可以在确认模板格式后，把生成边界改为：

```text
<|im_start|>assistant
<think>
我先用中文思考：
```

这叫 reasoning prefix prefill。它不是把英文思考翻译成中文，而是在模型生成前提供一个中文起点。它通常比只在前文声明语言更直接，但仍然是概率性引导，不是严格解码约束。

## 4. 不要猜真实提示词：直接检查渲染结果

排查这类问题时，仓库全文搜索只能回答“应用代码有没有这句话”，不能回答“模型最终看到了什么”。

对 vLLM 可以组合使用：

1. `/openapi.json`：确认当前服务真正接受哪些请求字段。
2. `/tokenize`：取得 Chat Template 渲染后的 token。
3. `/v1/chat/completions/render`：查看 Chat Completions 请求的渲染结果。
4. `/detokenize`：把 token 还原成可读提示词。
5. `/v1/chat/completions`：验证非流式和流式真实生成。

推荐最小对照：

```text
不传 effort
thinking_effort=low
reasoning_effort=low
```

比较三种请求的渲染文本，能直接判断参数是否进入 Chat Template。

## 5. 参数名必须在模型适配层映射

这次真实服务表现为：

| 请求 | 模板读取结果 |
|---|---|
| 不传 effort | 使用默认 `xhigh` |
| `thinking_effort=low` | 仍然使用默认 `xhigh` |
| `reasoning_effort=low` | 正确使用 `low` |

应用配置写的是“思考强度”，但不同模型服务可能使用不同字段。正确做法不是让 Runtime 上层到处判断供应商，而是在 Provider 请求边界做映射：

```text
内部统一配置 thinking_effort
          ↓ 模型适配层
qwen38  → reasoning_effort
qwen35  → thinking_effort
```

这样业务 Runtime 继续使用统一概念，协议差异留在适配层。模型切换、服务升级或别名变化时，只需重新验证并修改映射。

参数被静默忽略尤其危险：请求可能不会报错，却会回退到模板默认值。此次默认 `xhigh` 曾在 1200 个输出 Token 内全部用于思考，导致最终正文为空。参数兼容性不仅关系到推理质量，也关系到延迟、成本和正文空间。

## 6. 一套更稳的分层方案

### 应用层

- 按 locale 生成对应语言的输出合同。
- 同时约束 reasoning 和 final content。
- 保留 JSON、工具名和字段名等机器协议。
- 重组 System Prompt 时先移除旧合同，保证幂等注入。

### Provider 适配层

- 使用内部统一的强度配置。
- 根据实际模型服务映射参数名。
- 关闭思考时不发送 effort。
- 用请求载荷测试锁定字段行为。

### 模型服务层

- 把 Chat Template 自带的英文 reasoning 指令改成目标语言。
- 为所有支持的 effort 档位提供一致说明。
- 在 `<think>` 起点增加目标语言 prefill。
- 不随意翻译工具调用协议。

### 部署层

- 标准模型权重与定制 Chat Template 分开管理。
- 部署时覆盖模板、校验哈希、重启服务并做冒烟测试。
- 不同地区或语言使用不同模板制品或独立服务。

## 7. 验证矩阵

至少覆盖以下场景：

| 场景 | 需要确认 |
|---|---|
| 普通非流式回复 | reasoning 和正文语言、finish reason、正文非空 |
| SSE 流式回复 | reasoning delta 与 content delta 都能正确解析 |
| 工具调用 | reasoning 语言正确，工具名和参数没有被翻译 |
| `low/medium/xhigh` | 三档都进入模板，且说明语言正确 |
| 关闭思考 | 不发送 effort，模板正确闭合或跳过 reasoning |
| 长上下文 | 不因 reasoning 耗尽输出空间而丢失正文 |
| 模型切换 | 请求字段与目标服务 OpenAPI、模板变量一致 |
| 服务重启 | 模板确实重新加载，健康检查和真实生成都通过 |

不能把“模板文件改对了”当成完成。Chat Template 通常在服务启动时加载，修改后需要重启；也不能只看 `/health`，服务健康不代表模型语言与工具调用行为正确。

## 8. 部署与维护风险

直接修改权重目录适合快速验证，但不是长期资产管理方案。重新拉取权重、覆盖目录或迁移服务器后，模板修改可能消失。

更稳的发布流程是：

```text
拉取标准权重
→ 覆盖已版本化的业务 Chat Template
→ 校验哈希
→ 重启模型服务
→ render/detokenize 检查
→ 普通回复、流式回复、工具调用冒烟测试
```

还要明确模板的作用域。服务端 Chat Template 通常是整个模型实例的全局配置，不会自动根据每次请求的 locale 切换。中国业务版写死中文是可接受的简化，但国外业务应使用另一份模板或另一套模型服务。

## 9. 可复用排查清单

遇到“正文语言正确但思考语言错误”时：

1. 确认前端展示的是模型原始 reasoning，还是应用拼接或翻译后的内容。
2. 检查应用层语言合同是否明确覆盖 reasoning。
3. 全文搜索英文句子；搜不到时立即转向服务端模板。
4. 用 render/tokenize + detokenize 查看真实模型输入。
5. 对照测试不同 effort 参数名，确认哪个真正影响模板。
6. 检查 `<think>` 后的实际生成起点。
7. 修改模板前备份，只改自然语言说明，不破坏工具协议。
8. 重启模型服务，做流式、非流式和工具调用验证。
9. 检查输出 Token 预算，避免高强度 reasoning 挤掉正文。
10. 将定制模板纳入部署资产，避免服务器手工状态不可复现。

## 一句话记住

> 控制推理语言不能只靠业务 System Prompt：先看模型最终收到的渲染文本，再同时处理语言合同、effort 参数映射和 `<think>` 生成起点，并用真实流式与工具调用验证。

## 来源

- [Qwen38 中文思考问题完整交接](/Users/zhouchenghao/python_file/wj/ai-human/docs/handoff/2026-09-21-qwen38-chinese-reasoning-handoff.md)
- [AI Human 输出语言合同](/Users/zhouchenghao/python_file/wj/ai-human/app/core/localization/policy.py)
- [AI Human 语言合同归一化](/Users/zhouchenghao/python_file/wj/ai-human/app/core/localization/service.py)
- [AI Human reasoning 请求参数映射](/Users/zhouchenghao/python_file/wj/ai-human/app/agent_runtime/core/reasoning.py)
- [Company Qwen 请求封装](/Users/zhouchenghao/python_file/wj/ai-human/app/agent_runtime/core/company_qwen_client.py)
- [参数映射测试](/Users/zhouchenghao/python_file/wj/ai-human/tests/unit/test_mini_thinking_effort.py)

注：模型权重目录中的 Chat Template 位于远端服务器，不是当前 Git 仓库文件。部署环境或模型版本变化后，应重新检查真实模板和 OpenAPI 契约。
