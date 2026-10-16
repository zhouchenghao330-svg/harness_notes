# Hindsight Retain 写入流水线

> 组件 02：整体流程和既定三个分组的第 1 至 12 项已共同阅读并确认写入，本轮 Retain 清单已完成。下一组件为 03 记忆表示与持久化，尚未共同阅读。更新日期：2026-10-04。
>
> 本文先说明输入怎样经过 Retain 成为 bank 中的记忆，再记录本组件内部值得细看的操作。第一、二组和第三组第 9 至 12 项均已补充确认后的结论；第 11 项补充链接判断，第 12 项补充增量块复用及其运行边界。

## 1. Retain 整体做什么

Retain 接收原始内容及其上下文，把内容切成可处理的块，按配置提取事实及相关信息，生成向量、解析实体，再保存来源片段、事实及关联，使这些记忆能够被后续 Recall 和 Reflect 使用。【Code-verified】

一份输入可以生成多个 chunk，一个 chunk 也可以生成多条 fact。不同模式会改变这个关系，例如 chunks 模式直接使用块的内容，verbatim 模式保留原文并抽取相关信息；不能默认所有模式都先把原文改写成多条简短事实。【Code-verified】

## 2. 普通文本的整体流程

~~~text
输入文本 + context / timestamp / tags / metadata / document_id
                         |
                         v
              整理输入并确定本次处理配置
                         |
                         v
                   切成 source chunks
                         |
                         v
              构造 Prompt 和输出 Schema
                         |
                         v
              提取事实、时间、实体及关系
                         |
                         v
            加工向量输入并生成 Embedding
                         |
                         v
              整理事实并解析实体身份
                         |
                         v
        分批保存 chunks / facts / 实体关联 / 链接
                         |
                         v
                 返回写入结果与 usage
~~~

这是便于理解的数据处理顺序。普通新增写入使用生产者与消费者流水线，抽取与数据库写入可以交错执行；小文档也走这条路径。图中最后几步并不表示整个文档只执行一次数据库事务。【Code-verified】

五个阶段各自完成以下工作：

1. **整理输入**：处理文本与附带信息，解析 bank 配置和本次策略；已有文档还可能进入更新或增量处理分支。
2. **切块**：按文本或结构化内容的边界切分，同时保留块与输入内容的对应关系。
3. **提取事实**：根据模式构造 Prompt 和 Schema，从块中提取事实及时间、实体、因果引用等信息；部分模式跳过 LLM 抽取。
4. **准备检索信息**：在向量输入中补充日期和实体名，生成向量；整理抽取结果，并将实体名称解析为 bank 中的实体身份。
5. **保存结果**：保存来源块、事实和实体关联，建立相关链接。在普通数据库流式路径中，语义近邻链接还会在事实提交后补建。

以上阶段由当前源码确认。【Code-verified】实体和事实怎样在数据库中表示，将在组件 03 继续研究；这里关注 Retain 怎样生成和处理它们。

### 同步请求与后台请求

同步请求等待写入流水线完成后返回；后台请求先提交任务并返回 operation_id，由 Worker 后续执行 Retain。Python 方法名中的 `async` 表示异步函数，不能单凭这个名字判断请求是否已经进入后台队列。【Code-verified】

写入后还可能按配置提交归纳和维护任务。这些触发点先保留在整体认识中，具体归纳算法与 Worker 运行机制分别在组件 09 和 10 展开。【Code-verified，阅读安排】

## 3. 第一组 切块里面的操作

> 状态：已共同阅读并确认写入，2026-09-30。

### 3.1 初次切块选择边界

`retain_chunk_size` 的源码默认值是 3000，单位为字符，按 Python `len()` 计算；本次实际值由配置和策略决定。文本长度不超过限制时，整段作为一个 chunk。【Code-verified】

超长普通文本按以下优先级寻找当前文本中存在的分隔符：段落换行 `\n\n`、单行换行 `\n`、英文标点加空格（`. `、`! `、`? `、`; `、`, `）、空格、单个字符。

算法先选择最高优先级的现有分隔符，切出片段；短片段按顺序尽量合并进同一个 chunk，装不下时输出当前块；单个片段仍然过长则使用后面的分隔符继续递归切分。一个段落不一定独占一个块。【Code-verified】

例如限制为 12 字符，`AAAA\n\nBBBB\n\nCCCC` 输出 `AAAA\n\nBBBB`（10 字符）和 `CCCC`（4 字符）。这是运行当前源码函数得到的结果。

分隔符没有单独列出中文 `。`、`！`、`？`。例如限制为 10 字符，`今天讨论项目。明天实现功能。下周准备上线。` 输出：

~~~text
今天讨论项目。明天实
现功能。下周准备上线
。
~~~

因此，切分尽量利用已有边界，但不能保证完整句子，中文无换行、无空格文本可能最终按字符切。【Code-verified，运行示例】

### 3.2 对话和 JSON 结构保留

初次切块对超长内容进行格式判断：

| 内容形式 | 处理方式 |
| --- | --- |
| JSON 数组，成员都是对象 | 按成员边界打包为对话块；判断不强制要求 role 和 content |
| 单个 JSON 对象 | 在结构单元长度限制内完整保留，超出则按文本切分 |
| JSONL，至少两条非空行且每行都是 JSON 对象 | 按完整行打包，短行可合并 |
| 其他文本 | 使用递归文本切分 |

对话路径保留单轮边界，不强制把一问一答放在同一块。JSONL 路径保留单行边界。【Code-verified】

`retain_structured_chunk_size` 控制单个结构单元多长仍可完整保留，未配置时使用普通块长度。若普通长度为 3000、结构限制为 5000，单轮自身序列化长度为 4200，则可单独保留，即使超出普通块长度。单元自身超过结构限制时，对话和 JSONL 路径按两个限制的较小值继续切分；此时片段不保证是完整合法 JSON。【Code-verified】

### 3.3 抽取输出超限后的重新切分

初次切块控制输入大小。模型输出仍可能超过输出预算，触发 `OutputTooLongError` 后，`_extract_facts_with_auto_split()` 才进入重切分：

~~~text
当前 chunk 抽取输出超限
          |
          v
      拆为两个子片段
          |
          v
   两部分并行重新抽取
          |
          v
合并 facts 和 token usage
~~~

多成员 JSON 数组按成员数量从中间拆为两个数组；单成员对话数组可将该轮 content 对半拆，保留其他字段并重新包装；普通文本在中部附近寻找英文句末或段落边界，找不到则从中间拆。此重试函数没有单独的 XML 或 JSONL 结构保留分支。【Code-verified】

子片段继续使用原 chunk_index，它们属于原来源块。超限后可以递归拆分，但去掉首尾空白后长度不超过 500 字符，或无法继续缩小时，不再拆分，记录 warning 并返回空事实列表。这个分支会放弃该子块的事实抽取，不直接导致整个 Retain 失败；其他异常另有传播与重试路径。500 字符下限只在输出超限后的重切分中生效，正常小块照常抽取。【Code-verified】

### 3.4 输入格式与 XML 问题

普通 Retain 的 HTTP 请求以 JSON 表达 items 和参数，正文 content 可以是普通字符串，也可以是文本与图片组成的内容块列表。HTTP 层将块列表转为正文及附件引用，再交给下游处理；另有文件上传、转换后进入 Retain 的路径。【Code-verified】

记忆来源包括对话、文档、笔记和 Agent 执行记录；调用方决定送入哪些内容。请求外层是 JSON，不要求 content 正文也是 JSON。XML 可以作为正文字符串送入；短 XML 整段处理，长 XML 使用普通文本切分，当前切块器没有 XML 解析与标签配对保护。【Code-verified + Docs-described】

仓库 coding-agent 集成将会话序列化为每轮一行的 JSONL；注释说明这样方便追加，也便于利用服务端的结构单元切块能力。JSON/JSONL 属于实际使用场景。【Code-verified】据此可以推断这些格式得到优先覆盖，但没有确认作者不实现 XML 的具体理由，不能把该推断当作作者的设计声明。【Inferred】

### 3.5 Overlap 与跨块上下文

当前普通文本切分对应 `chunk_overlap=0`，对话和 JSONL 打包也没有自动复制相邻内容；没有看到可配置的 Retain chunk overlap 参数。【Code-verified】

~~~text
没有 overlap
块 1：张三负责数据库迁移。
块 2：他还负责发布后的检查。

有 overlap 的示意
块 1：张三负责数据库迁移。
块 2：张三负责数据库迁移。他还负责发布后的检查。
~~~

完整轮次边界和调用方提供的 context、日期可以帮助理解，但不等于自动补上相邻块正文。跨块代词、人物关系或因果关系可能因为上下文切开而抽取不完整；后续建链接不能保证恢复漏抽内容。【Code-verified + Inferred，影响边界】

零重叠可能减少重复抽取与模型成本，但源码没有明确说明采用该设置的动机，暂作为推断记录。【Inferred】

### 3.6 本组源码证据

路径相对 `hindsight-api-slim/hindsight_api/`，集成路径相对仓库根目录：

- `config.py`：`DEFAULT_RETAIN_CHUNK_SIZE` 和 retain 切块配置。
- `engine/retain/fact_extraction.py`：`iter_chunks()`、`_iter_recursive_splits()`、`_iter_conversation_chunks()`、`_iter_jsonl_chunks()`、`_split_chunk_for_output_retry()`、`_extract_facts_with_auto_split()`、`_build_user_message()`。
- `api/http.py`：`MemoryItem`、`RetainRequest`、`canonicalize_item_content()`、`api_file_retain()`。
- `hindsight-integrations/coding-agents/src/core/chat.ts`：`renderSessionJsonl()` 和 `ingestChats()`。
- `hindsight-docs/docs/developer/api/retain.mdx`：对话、文档、笔记输入说明。

## 4. 第二组 事实抽取里面的操作

> 状态：已共同阅读并确认写入，2026-09-30。

本组研究一个 chunk 怎样变成事实，包括 Prompt、事实粒度、人物归属、时间和分类标签。事实正文、内置类型及配置后的自定义标签通常在同一次模型调用中一起返回，没有额外调用独立分类模型。【Code-verified】

### 4.1 处理流程图

~~~text
第一组产出的 chunk
  + 日期 / context / metadata
                         \
                          \
Bank 配置 ----------------+
  抽取模式                  |
  retain_mission            v
  自定义标签规则      构造 Prompt + 输出 Schema
                            |
                            v
                       一次 LLM 调用
                            |
                            v
                    返回 facts 数组
                    每条可以包含：
                    +--------------------------+
                    | what / when / where     |
                    | who / why               |
                    | fact_type / fact_kind   |
                    | occurred_start / end    |
                    | entities / 因果引用      |
                    | labels（配置后才有）     |
                    +--------------------------+
                            |
                            v
                     代码解析和整理
                     - 拼接事实正文
                     - assistant -> experience
                     - 处理发生日期和因果引用
                     - 检查标签值
                     - labels -> key:value 实体
                            |
                            v
                 绑定来源 chunk 和输入信息
                 必要时回填 verbatim 原文
                 添加时间顺序偏移
                 tag=true 的标签复制到 tags
                            |
                            v
                  第二组产出的事实列表
                            |
                            v
                  第三组操作（第 9 至 12 项均已共同阅读并写入）
~~~

上图描述普通 LLM 抽取路径，不表示每次所有可选字段都会出现。chunks 模式绕过 LLM 抽取，一个块直接成为一个记忆单元，也不会通过这段模型调用自动生成分类标签。【Code-verified】

### 4.2 Prompt 与模型请求

请求由三部分组成：System Message 说明抽取规则；User Message 提供当前材料；Response Schema 定义结构化返回格式。

~~~python
llm_config.call(
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_message},
    ],
    response_format=response_schema,
    # 另有温度、输出预算、严格 Schema 与重试设置
)
~~~

默认 concise Prompt 的开头原文是：

> Extract SIGNIFICANT facts from text. Be SELECTIVE - only extract facts worth remembering long-term.

后续规则说明五个描述维度、人物指代、类型分类、时间、实体与可选因果关系。默认要求保留输入内容的语言；配置输出语言时会加入相应指令。【Code-verified】

以下是依照实际 User Message 模板构造的教学示例，不是一次真实模型运行的记录：

~~~text
Extract facts from the following chunk.

Chunk: 1/1
Event Date: Wednesday, September 30, 2026 (2026-09-30T10:00:00+08:00)
Context: 用户张三与助手的对话

Content:
用户：我昨天去了杭州参加技术会议。以后请一直用中文回复。
助手：我已经把默认回复语言改成中文了。
~~~

传入 metadata 时会加入 Metadata 部分。配置 retain_mission 时，任务说明放在 User Message 最前面，并声明其优先于一般抽取指导；源码解释这样安排便于缓存通用 System Prompt。【Code-verified】

这条抽取请求使用当前块及附带信息，没有自动读取旧记忆或相邻块正文。context 是调用方提供的说明。【Code-verified】

### 4.3 事实粒度与抽取模式

一句话对应几条事实主要由模型依据 Prompt 判断，没有固定的一句话一条规则。concise 模式要求保留个人信息、偏好、重要事件、计划、约束及有意义的感受和观察，跳过问候、客套、纯过程性话语，尽量合并相关陈述。【Code-verified，Prompt 要求】

Prompt 要求不反复提取当前材料中的重复信息。这不能证明它查询了数据库或完成了全库事实去重。筛选规则是模型指导，不保证每次选择和措辞完全相同。

| 模式 | 主要行为 |
| --- | --- |
| concise，默认 | 筛选重要内容，简洁表达并合并相关陈述 |
| verbose | 尽量保留细节，描述更完整 |
| custom | 在基础抽取规则中加入自定义指导；未配置指导时退回 concise |
| verbatim | 保留块的原文，仍让模型提取实体、时间等信息 |
| chunks | 跳过 LLM，一个块直接成为一个记忆单元 |

以上由模式分支和 Prompt 模板确认。【Code-verified】

对 4.2 的输入，可能提取出“张三于 2026-09-29 去杭州参加技术会议”“张三希望助手始终使用中文回复”“助手已修改默认回复语言”等事实。此处只是示意，实际数量和措辞由模型决定。

模型先返回 what、when、where、who、why 等结构化字段，代码再转换为下游 Fact。当前普通解析路径的正文拼接使用 what、when、who、why，没有把 where 单独加入拼接；地点若包含在 what 中则随正文保留。因此，不能假设所有 Schema 字段都原样进入最终正文。【Code-verified】

### 4.4 说话者与内置分类

人物指代主要依靠正文中的身份标记和 context。当前普通 Retain 路径不会把 Bank 显示名称自动当作说话者；明确身份应由材料或 context 提供。【Code-verified】

三种分类要分开理解：

| 字段 | 作用 | 自定义能力 |
| --- | --- | --- |
| fact_type | 区分外部事实与助手经历 | 模型返回 world / assistant，下游为 world / experience |
| fact_kind | 区分具体事件与持续状态、偏好，控制时间处理 | 内置 event / conversation 等解析判断；当前不作为最终 Fact 独立字段保存 |
| labels | 为事实赋予业务分类 | 可定义维度、说明、取值方式及候选值 |

用户的偏好、规则、纠正、约束和个人事件归为 world；助手实际做过、尝试过或观察到的事情由模型标为 assistant，再转换为 experience。分类依靠输入和模型判断，不验证助手声称执行的操作是否真的成功。【Code-verified + 作用边界】

例如“用户希望始终使用中文回复”可以同时是 world、conversation，并具有自定义标签 topic:交流偏好。topic 是示例分类维度，不是所有事实默认具有的字段。

### 4.5 时间解析与顺序偏移

Prompt 要求使用输入的 Event Date 作为相对日期基准，并将正文中的相对日期也改成明确日期。参考日期为 2026-09-30 时，“昨天去了杭州”应解析到 2026-09-29。仅提到某年或某月时，Prompt 要求记录整个时期的范围。【Code-verified，Prompt 要求】

occurred_start/end 描述实际发生时间；mentioned_at 使用输入内容的参考日期。解析路径只为 event 设置发生时间，conversation 描述持续状态或偏好，即使来自聊天，也通常没有发生日期。【Code-verified】

模型未给出事件开始时间时，代码有相对日期兜底，主要匹配 yesterday、last week 等英文表达；中文相对日期仍主要依靠模型。【Code-verified】

转换后，代码在当前抽取结果列表中按序号 i 添加 i × 0.01 秒偏移，作用于已有的发生时间和提及时间。这是程序添加的顺序信息，不能理解为事件真实相隔 10 毫秒，也不证明所有事件的真实时间顺序已被还原。【Code-verified，作用边界】

### 4.6 自定义标签与自动分类

Bank 的 entity_labels 定义分类维度。维度可以叫 topic，也可以是业务领域、任务类型等，并可同时配置多个维度。模型依据每条事实填写 labels，因此一份文档中的不同事实可以有不同分类。【Code-verified】

固定候选值配置示例：

~~~json
{
  "entity_labels": [
    {
      "key": "topic",
      "description": "这条事实主要涉及哪个领域",
      "type": "value",
      "optional": true,
      "tag": true,
      "values": [
        {"value": "交流偏好"},
        {"value": "工作任务"},
        {"value": "个人生活"}
      ]
    }
  ]
}
~~~

代码将维度说明和候选值加入 Prompt，并动态构造 Schema。模型自动选择值，调用方无需逐条标注；固定枚举之外的值会被跳过。optional 为 true 时，不适用的单选标签可以为空。【Code-verified】

| 标签类型 | 模型输出 |
| --- | --- |
| value | 一个固定候选值 |
| multi-values | 多个固定候选值 |
| text | 一个自由文本值 |
| multi-text | 多个自由文本值 |
| map | 按配置字段提取结构化信息 |

改为 text 或 multi-text 后，不用预列全部值，模型可根据内容生成“回复语言偏好”“数据库迁移”等值。但仍需要先定义分类维度及指导。默认 entity_labels 为空，系统不会默认发明整套分类体系；自由文本也不保证把“交流偏好”和“沟通偏好”统一成同一名称。【Code-verified + 作用边界】

### 4.7 分类结果如何进入实体和 tags

~~~text
模型输出：labels = {"topic": "交流偏好"}
                      |
                      v
              topic:交流偏好
                      |
                      +--> 加入该事实的 entities
                      |
                      +--> 标签组 tag=true 时加入 tags
~~~

这段转换发生在第二组。模型输出的 labels 被处理为实体字符串，配置 tag=true 时再投影为 tags。普通实体名称同样可以由模型抽取，例如 user、杭州；这些名称的数据库身份解析属于第三组。【Code-verified】

例子的下游内容可以理解为：

~~~text
正文：用户希望始终使用中文回复。
类型：world
entities：[user, topic:交流偏好]
tags：[topic:交流偏好]  # 配置 tag=true 时
~~~

entities_allow_free_form 默认为 true，标签和普通实体可共同抽取；设为 false 时，抽取结果仅保留标签实体。调用方手工传入的 tags 与模型派生标签是不同来源，不能把所有 tags 都当作模型分类结果。【Code-verified】

### 4.8 本组源码证据

路径相对 `hindsight-api-slim/hindsight_api/`：

- `engine/retain/fact_extraction.py`：Prompt 常量、ExtractedFact 和 Fact 模型、`_build_extraction_prompt_and_schema()`、`build_chunk_prompt_parts()`、`_build_user_message()`、`_extract_facts_from_chunk()`。
- 同文件：`extract_facts_from_contents()`、`_collapse_to_verbatim()`、`_infer_temporal_date()`、`_add_temporal_offsets()`、`_inject_label_tags()`。
- `engine/retain/entity_labels.py`：LabelGroup、EntityLabelsConfig、`build_labels_model()` 和标签值检查辅助函数。
- `config.py`：默认抽取模式、entity_labels 与 entities_allow_free_form 初始值。
- 仓库文档 `hindsight-docs/docs/developer/api/memory-banks.mdx` 的 entity_labels 小节：分类类型与配置说明。

## 5. 第三组 抽取结果进入写入前后的操作

> 状态：第 9 至 12 项均已共同阅读并确认写入。第 9 项写入日期为 2026-10-03；第 10 至 12 项写入日期为 2026-10-04。

9. **合并并解析实体名称**：已共同阅读并写入，见下方 5.1。涵盖名称处理、实体 ID、名称与共现评分、时间加分，以及与 Consolidation 的区别。
10. **加工用于 Embedding 的文字**：已共同阅读并写入，见下方 5.2。涵盖日期与实体名的追加、事实正文与向量输入的区别，以及与实体身份解析的边界。
11. **确定事实之间的链接**：已共同阅读并写入，见下方 5.3。包含四类关系的独立判断、门槛与权重、存储与展示，以及普通和流式路径的时机。
12. **识别可以复用的旧块**：已共同阅读并写入，见下方 5.4。包含同编号哈希比较、四类变化处理、旧记忆清理、重抽与并发保护。

代码入口：`entity_processing.py` 的 `_prepare_facts_for_entity_processing()`、`resolve_entities()`；`embedding_processing.py` 的 `augment_texts_with_dates()`；`link_creation.py`、`link_utils.py`；`orchestrator.py` 的 `_process_extracted_facts()`、`_insert_facts_and_links()`、`_run_final_semantic_ann()`、`_try_delta_retain()`。

这三组按学习顺序组织。第三组中的增量判断在实际执行中可以发生于重新抽取之前，不能把清单编号当作严格的运行时顺序。

### 5.1 第 9 项 合并并解析实体名称

> 状态：已共同阅读并确认写入，2026-10-03。本项解释 Retain 中的实体身份解析；Consolidation 只用于对照用户实际看到的行为，其完整流程仍属于组件 09。

#### 5.1.1 这一项的输入和输出

第二组得到事实和实体名称。本项把名称匹配到当前 bank 的实体记录，得到可以稳定引用的实体 ID，并建立事实与实体的关联。【Code-verified】

```text
第二组输出
Fact：Alice 在 Acme 公司使用 Python 编程。
Entities：[Alice, Acme, Python]
                 |
                 v
合并调用方指定的实体名称
                 |
                 v
清理空白、去掉无效名称、同一事实内去重
                 |
                 v
在当前 bank 中查找已有实体候选
                 |
                 v
名称检查 + 共现加分 + 时间加分
        /                         \
       v                           v
满足条件：复用已有 ID       不满足：创建新实体 ID
        \                         /
                 v
Fact ID -> Entity IDs
                 |
                 v
保存事实与实体关联，积累实体共现关系
```

实体 ID 用来表示数据库中同一个节点；名称是这个节点的显示和匹配信息。例如 `Alice` 和 `Alice Chen` 能否引用同一个 ID，要靠解析规则决定。ID 本身不能判断两个人是否相同，也不能自动解决别名、翻译或者同名问题。【Code-verified】

只用名称作为主键并非做不到，但需要承担名称唯一、名称变化、多个写法等约束。这里把稳定身份和可变化的名称分开，因此事实关联、实体关系和后续检索可以引用 ID。【Inferred，设计解释】

#### 5.1.2 实体名称从哪里来

1. **模型抽取的实体**：来自事实的 `entities`，默认允许解析到已有实体。
2. **调用方指定的实体**：按原始输入的 `content_index` 找到该内容的实体配置，补入这份内容产生的各条事实。
3. **配置产生的标签实体**：例如 `topic:工作`，来自第二组的分类转换；解析时按标签规则处理。【Code-verified】

模型名称和调用方名称先按小写形式合并。随后再压缩连续空白、去掉首尾空白，丢弃空名称和异常过长的名称，并在同一事实内再次按小写形式去重。保留名称的显示大小写；用于比较的形式可以转为小写。【Code-verified】

调用方设置 `resolve_entities=false` 时，它指定的名称走忽略大小写的精确查找或创建，跳过模糊解析。这个设置不会整体关闭模型抽取实体的解析；如果双方给出同一名称，则保留调用方要求的严格行为。配置识别出的标签实体也只做忽略大小写的精确匹配，避免相似标签值被错误归并。【Code-verified】

#### 5.1.3 已有实体怎样被选中

当前实体解析没有调用 LLM，也没有用实体 Embedding 判断名称的语义是否一致。【Code-verified】

候选查找支持 `full` 和 `trigram` 等路径：前者加载 bank 内实体并检查名称，后者使用 PostgreSQL 的 `pg_trgm` 相似名称查询；扩展不可用时有回退路径。候选被查到之后，还要通过名称检查和最终评分，不能把进入候选集理解为已经合并。【Code-verified】

普通模糊匹配大体分为两层：

1. **名称门槛**：检查名称的 trigram 集合相似度，默认最低值为 `0.3`；还会做名称词项兼容检查，防止某个长单词的相似掩盖另一个完全不同的词项。
2. **综合评分**：名称最多贡献 `0.5`，共现最多贡献 `0.3`，时间最多贡献 `0.2`。选择得分最高的候选，得分四舍五入到六位小数后达到 `0.6` 才复用其 ID。【Code-verified】

名称 trigram 集合一致时有直接记 `1.0` 的分支，例如仅大小写、部分标点或装饰不同。它不会要求再提供共现或时间证据。普通分支的公式为：【Code-verified】

```text
总分 = 0.5 * SequenceMatcher 名称相似度
     + 0.3 * 加权共现重合比例
     + 0.2 * 时间接近程度

总分 >= 0.6：复用最高分候选的实体 ID
否则：进入新实体创建路径
```

本批次准备创建的新名称之间，也有名称相似归并路径，默认 trigram 门槛为 `0.5`，并受词项兼容检查等条件限制。标签和调用方要求按字面处理的名称不参与这种模糊归并。它与匹配已经存在的历史实体不是同一个评分过程。【Code-verified】

因此，`乳糖不耐受` 和 `lactose intolerance` 即使语义一致，当前实体解析也没有专门的翻译或向量语义判断来保证它们共用 ID。后续事实向量检索是否能找到相关内容，是另一个问题；找到语义相关事实不等于实体 ID 已合并。【Code-verified】

#### 5.1.4 共现关系怎样保存和读取

共现的单位是**同一条事实**。同一份文档中分属不同事实的两个实体，不会仅因为文档相同就产生这条共现记录。【Code-verified】

```text
事实 F1 的实体 ID 集合：[Alice, Acme, Python]
                  |
                  v
unit_entities：F1 -> 三个实体 ID
                  |
                  v
对同一事实内的不同实体两两配对

Alice ----- Acme
   \         /
    \       /
      Python

entity_cooccurrences 保存：
Alice <-> Acme
Alice <-> Python
Acme  <-> Python
```

`entity_cooccurrences` 存储实体 ID 对、`cooccurrence_count` 和 `last_cooccurred`。实体 ID 对按固定顺序存储，避免把 A-B 与 B-A 当成两条不同关系。写入时先从事实与实体的关联生成配对，积累待更新记录，Retain 事务提交后再刷新统计。【Code-verified】

**计数不严格等于事实条数**：一次批量关联调用中，相同实体对会先去重；提交后按积累的配对记录增加计数。因此不能直接说“十条事实中一起出现十次，计数就一定加十”。【Code-verified】

解析新名称时，代码读取候选实体两侧的共现边，构建该候选历史共现伙伴的名称集合，再与本次同一事实内其他实体的名称集合比较。评分没有直接拿 `cooccurrence_count` 做频率权重，而是看伙伴是否重合，以及伙伴是否过于普遍。【Code-verified】

```text
本次 Alice Chen 的其他实体：{Acme, Python}
已有 Alice 的历史伙伴：    {Acme, Python, ...}

匹配到的伙伴 = 两个集合的交集

每个伙伴的权重 = 1 / sqrt(max(该伙伴的不同共现伙伴数, 1))

共现加分 = 0.3 * 匹配伙伴的权重之和 / 本次其他实体数量
```

例如一个实体与 100 个不同实体共现，每次重合只贡献 `0.1` 的权重；只与一个实体共现时贡献 `1`。代码降低 `user` 等广泛出现的实体的影响，避免只因都与 user 出现就把两个不同的人归为一个。这里的“不同伙伴数”不是某一对实体出现了多少次。【Code-verified】

这些是实体之间的共现关系；第 11 项将讨论的事实之间的语义、时间等链接，需要另行区分。

#### 5.1.5 Alice 与 Alice Chen 的计算例子

以下是按当前源码计算的教学例子，**不是对用户那次 Retain 的实际参数或评分的重放**。假设 bank 中已有 Alice、Acme、Python，三者只在一条事实中一起出现；本次 Alice Chen 的同一事实也包含 Acme、Python，并且只考虑 Alice 这个候选。【Code-verified，源码函数计算示例】

**名称部分**：

```text
alice 与 alice chen

trigram 相似度：6 / 11 = 0.545455
通过默认 0.3 的名称门槛，词项也兼容

SequenceMatcher 相似度：2 * 5 / (5 + 10) = 0.666667
名称加分：0.666667 * 0.5 = 0.333333
```

trigram 用字符三元组集合计算交并比，词边界有填充；它在此处用于先过滤名称。最终名称加分使用 SequenceMatcher，不能把这两个相似度混用。【Code-verified】

**共现部分**：已有三角形关系中，Acme 的不同伙伴是 Alice、Python，Python 的不同伙伴是 Alice、Acme，所以两者 degree 均为 `2`。【Code-verified，示例条件】

```text
本次其他实体：{Acme, Python}
两个伙伴都与历史 Alice 重合

每个伙伴权重：1 / sqrt(2) = 0.707107
共现加分：(0.707107 + 0.707107) / 2 * 0.3 = 0.212132

名称 + 共现 = 0.333333 + 0.212132 = 0.545465
```

不加时间分时，结果低于 `0.6`，不会在这个历史候选评分分支里复用 Alice 的 ID。因此“名字部分重合，又都与 Acme、Python 出现”并不保证归并。【Code-verified，示例条件】

#### 5.1.6 时间部分怎样加分

当前事实的参考日期优先使用 `occurred_start`，没有时使用 `mentioned_at`。候选实体的 `last_seen` 由关联事实的参考日期累积更新，取较晚日期；它不一定是上传时间。【Code-verified】

```text
days_diff = abs(当前事实参考日期 - 候选实体 last_seen)，单位为天

有两个日期，且相差小于 7 天：
    时间加分 = (1 - days_diff / 7) * 0.2

缺少日期，或相差达到 7 天：
    时间加分 = 0
```

日期差使用带小数的天数，不是先取整；过去或未来方向都按绝对距离计算。没有时间分时也不会把另外两项的权重重新放大。【Code-verified】

沿用上面的 Alice 示例，名称与共现合计 `0.545465`：

| 日期差 | 时间加分 | 总分约值 | 是否达到 0.6 |
| --- | --- | --- | --- |
| 0 天 | 0.200000 | 0.745465 | 是 |
| 3 天 | 0.114286 | 0.659751 | 是 |
| 5 天 | 0.057143 | 0.602608 | 是 |
| 6 天 | 0.028571 | 0.574037 | 否 |
| 至少 7 天或缺日期 | 0 | 0.545465 | 否 |

表中结果只适用于这些教学条件。名称门槛没有通过时，时间和共现不会把候选救回来；存在其他候选时，还要比较最高分。【Code-verified】

#### 5.1.7 用户实验里为什么观察仍然合并了

用户实际输入 Alice 与 Alice Chen 的例子后，看到 Retain 阶段两个人物实体仍分开，而 Consolidation 返回了对已有观察的 `updates`：【用户提供的运行输出与截图】

```json
{
  "creates": [],
  "updates": [
    {
      "text": "Alice Chen在Acme公司使用Python编程。",
      "observation_id": "d52a1d7a-90cf-43f0-bbd4-9983f7879cce",
      "source_fact_ids": [
        "a2cd1d9e-0a68-4dd8-aeea-52153f77b9aa"
      ],
      "reason": "The new fact provides the full name 'Alice Chen' for the same person and role already captured in obs d52a1d7a, so I updated the observation to include the full name and merged the new fact as evidence."
    }
  ],
  "deletes": []
}
```

观察历史页显示：观察文本由 Alice 改成 Alice Chen，原来源事实保留，新来源事实加入，来源事实总数变为两条，提及时间也更新。这是模型在 Consolidation 中更新观察及其证据的行为。【用户提供的运行输出与截图】

```text
Retain 实体解析                         Consolidation

Fact 1 -> Alice 实体 ID                 Fact 1 + Fact 2 + 已有观察
Fact 2 -> Alice Chen 实体 ID                       |
                                                  v
可能仍是两个实体节点                    LLM 判断可以更新同一条观察
                                                  |
                                                  v
                                       Observation 文本改为 Alice Chen
                                       来源事实保留并补充为两条
```

**更新观察不代表实体 ID 已合并，也不代表原事实被改写或删除。** 模型把两个事实视为同一人的证据，是这次归纳的判断；它本身也不能证明现实中两个人一定相同。Consolidation 的完整更新、删除和证据管理流程留在组件 09 展开，本项只确认两种处理发生在不同层次。

#### 5.1.8 本项源码证据

路径相对 `hindsight-api-slim/hindsight_api/`。上述规则对应当前本地源码；用户 UI 截图与模型输出用于说明其实际运行结果，教学计算不冒充该次运行的 trace。

- `engine/retain/entity_processing.py`：`_prepare_facts_for_entity_processing()`、`resolve_entities()`，合并模型与调用方实体并传入参考日期。
- `engine/retain/link_utils.py`：`_normalize_entity_name()`、`_prepare_entities_for_resolution()`、`resolve_entities_only()`，名称清理、同事实实体上下文和事实关联映射。
- `engine/entity_resolver.py`：`resolve_entities_batch()`、`_resolve_from_candidates()`，候选查找、名称门槛、综合评分及创建路径。
- 同文件：`_build_cooccurrence_index()`、`_cooccurrence_weight()`、`_link_units_to_entities_batch_impl()`、`flush_pending_stats()`，读取共现伙伴、构建实体对及提交后统计更新。
- `config.py`：`DEFAULT_ENTITY_MERGE_MIN_SIMILARITY = 0.3`、`DEFAULT_ENTITY_INTRABATCH_MERGE_SIMILARITY = 0.5`。综合评分分支的 `0.6` 门槛见 resolver。

### 5.2 第 10 项 加工用于 Embedding 的文字

> 状态：已共同阅读并确认写入，2026-10-04。核心结论：事实正文与计算事实向量的输入可以不同；当前代码把日期和实体名称追加到向量输入中，正文仍单独保存。

#### 5.2.1 同一条事实有正文和向量

第二组抽取事实后，`augment_texts_with_dates()` 构造一份新的字符串列表。`generate_embeddings_batch()` 将这些字符串送给 Embedding 后端，生成与事实逐项对应的向量。加工函数不修改原事实正文。【Code-verified】

```text
第二组得到的 Fact
        |
        +--> 原事实正文 ----------------------> 保存正文
        |
        +--> 正文 + 日期 + 实体名称
                         |
                         v
                   Embedding 模型
                         |
                         v
                      事实向量 ---------------> 保存向量
```

保存时事实记录分别携带 `text=fact.fact_text` 和 `embedding`。页面展示的正文不需要包含用于向量计算的所有补充文字。【Code-verified】

#### 5.2.2 Alice 的实际加工例子

教学输入如下，不代表用户那次 Retain 的实际请求日期：

```text
fact_text：Alice 在 Acme 公司使用 Python 编程。
occurred_start：null
occurred_end：null
mentioned_at：2026-10-01
entities：[Alice, Acme, Python]
```

当前源码函数输出的 Embedding 输入为：【Code-verified，已执行源码加工函数核对】

```text
Alice 在 Acme 公司使用 Python 编程。 (happened in October 2026) [Alice, Acme, Python]
```

原正文仍然是：

```text
Alice 在 Acme 公司使用 Python 编程。
```

这一步是 Python 字符串拼接，不会让 LLM 再翻译一次事实。英文 `happened in` 来自固定模板；实体部分直接追加抽取出的名称字符串，包括可能存在的 `key:value` 标签。【Code-verified】

#### 5.2.3 日期和实体怎样补进去

**日期规则**：优先使用 `occurred_start`，没有时回退到 `mentioned_at`。当前 `_format_readable_date()` 输出月份和年份，默认英文月份环境下例如 `October 2026`；精确日期保留在事实的结构化字段里。【Code-verified】

```text
单个日期：
(happened in October 2026)

存在 occurred_end，且与 occurred_start 不同：
(happened from October 2026 to November 2026)

两个可用的代表日期都没有：
不追加日期部分
```

代码用可读日期为时间相关查询提供向量输入线索，例如“十月份做了什么”。这表达了代码的设计意图，实际召回效果仍取决于 Embedding 模型和查询内容。【Code-verified，设计意图；效果未作评测】

**回退日期的含义要区分**：`mentioned_at` 表示内容被提及的参考日期，不自动等于事件实际发生日期。固定的 `happened in` 模板本身没有表达这一差别；不能因为追加了月份就把它当成新的事实证据。【Code-verified】

**实体规则**：有实体名称时，使用逗号连接后追加在方括号里；没有实体时，不追加这部分。这样实体名称和配置产生的标签也能参与事实向量的计算。它们提供额外的匹配线索，但不保证任意模型都能改善召回。【Code-verified，设计意图；效果未作评测】

#### 5.2.4 与第 9 项的区别和实际执行顺序

| 操作 | 处理对象 | 结果 |
| --- | --- | --- |
| 第 9 项 实体解析 | Alice、Alice Chen 等名称及其匹配上下文 | 复用或创建实体 ID |
| 第 10 项 Embedding 输入加工 | 整条事实及其日期、实体名称 | 生成事实向量 |

第 10 项调用的是 Embedding 后端，得到一组数值向量；这一步不让事实抽取 LLM 再输出一份 JSON，也不使用这些事实向量判断 Alice 与 Alice Chen 是否共用实体 ID。【Code-verified】

编号是学习顺序，不能据此断言第 9 项总在第 10 项之前执行。在核对的 `_extract_and_embed()` 路径里，先用抽取结果中的实体名称生成事实向量，形成处理后的事实，随后再进入数据库实体身份解析。因此追加到向量输入的是名称字符串，而不是已经解析好的实体 ID。【Code-verified】

#### 5.2.5 本项源码证据

路径相对 `hindsight-api-slim/hindsight_api/`：

- `engine/retain/embedding_processing.py`：`augment_texts_with_dates()`、`generate_embeddings_batch()`，日期和实体追加及向量后端调用。
- `engine/memory_engine.py`：`_format_readable_date()`，当前输出月份和年份。
- `engine/retain/orchestrator.py`：`_extract_and_embed()`、`_process_extracted_facts()`，抽取、向量生成，以及事实与向量的逐项对应。
- `engine/memories/base.py`：构造 `FactRecord` 时分别传入事实正文与向量。

### 5.3 第 11 项 确定事实之间的链接

> 状态：已共同阅读并确认写入，2026-10-04。包括四类关系的来源、独立判断方式、权重、持久化和展示差异，以及普通写入与流式写入的时机。

#### 5.3.1 四类关系分别判断

事实保存之后，需要有路径找到与它相关的其他事实。语义、时间、因果和实体关系分别提供这种关联；它们会影响后续检索和图展示。【Code-verified】

```text
Fact A                         Fact B
   |                              |
   +------ 内容相似：Semantic ------+
   +------ 时间相关：Temporal ------+
   +------ 存在因果：Causal --------+
   +------ 共享实体：Entity --------+
```

**四类关系各自判断，没有把四项加权成总分后统一决定是否连线的公式。** 同一对事实可以同时存在多种关系。每种关系需要满足本类型的规则，并经过邻居数量等限制；因果和实体关系也不都使用数值门槛。【Code-verified】

例如以下教学条件：

| 类型 | 条件 | 结果 |
| --- | --- | --- |
| Semantic | 相似度 0.65，低于默认门槛 0.7 | 不生成语义链接 |
| Temporal | 被对应时间邻居规则选中 | 生成时间链接 |
| Causal | 没有有效的模型因果引用 | 不生成因果链接 |
| Entity | 两条事实关联同一个 Alice 实体 | 存在共享实体关系，可用于检索和图展示 |

这里语义门槛没有通过，不妨碍时间或实体关系成立。各类链接上的权重属于该类关系，不合并为一次建边总分。【Code-verified】

#### 5.3.2 语义链接使用事实向量

语义链接计算第 10 项生成的事实向量之间的余弦相似度。当前源码默认门槛为 `0.7`，对应配置 `semantic_link_min_similarity` 和环境变量 `HINDSIGHT_API_SEMANTIC_LINK_MIN_SIMILARITY`；实际值由生效配置决定。相似度达到门槛后，仍受候选邻居数量限制。【Code-verified】

```text
A：Alice 在 Acme 使用 Python 编程。
B：Alice 用 Python 开发了一个工具。

A 的事实向量 + B 的事实向量
              |
              v
         计算余弦相似度
              |
              v
达到门槛且进入邻居结果 -> Semantic 链接
```

相似度是向量匹配分数，不是两条事实相同的概率。建立链接后，两条事实仍独立保存，没有因此合并正文或事实 ID。【Code-verified】

普通路径同时处理两类邻居：

- **库内已有事实**：ANN 近邻搜索按 bank 和当前事实的 `fact_type` 过滤。普通函数默认每个种子取最多 50 个候选，再按相似度门槛筛选。
- **本批新事实**：在内存中计算余弦相似度，排除自身和无效向量，并取达到门槛的最高分邻居；该批内函数没有按 `fact_type` 分组，默认 `top_k=50`。【Code-verified】

流式路径可以延后到全部小批次提交后统一运行最终 ANN，此处每个种子使用 `top_k=20`。近似搜索和邻居上限意味着不能保证枚举所有达到门槛的事实对。【Code-verified】

#### 5.3.3 时间链接的两条路径

时间链接读取事实保存的 `event_date`。该字段可以来自发生日期，也可以回退到提及日期；两个参考日期接近不证明它们属于同一个真实事件。【Code-verified】

当前本地 PostgreSQL 实现需要区分：

| 路径 | 实际规则 |
| --- | --- |
| 本批新事实之间 | 按 `fact_type` 分组，比较日期差，默认不超过 24 小时 |
| 从数据库寻找邻居 | 同 bank、同 `fact_type`，在时间前后寻找最近候选，再取最近的结果；实际 SQL 没有硬性的 24 小时过滤 |

因此，“所有时间链接都限定在一天内”不是当前代码支持的结论。函数和注释提到 24 小时，必须同时核对 `fetch_temporal_neighbors()` 的实际查询；库内邻居可能超过一天。【Code-verified】

```text
weight = max(0.3, 1 - 日期差小时数 / 24)

相差 6 小时：weight = 0.75
相差很久：weight 最低为 0.3
```

这里的权重不是统一建边门槛。候选还会按每个起点保留最多 20 条时间链接；它是写入函数的起点预算，不保证图上每个节点的总连接数最多 20，因为其他节点还可能连接到它。没有可用日期的事实不作为本批时间链接的日期种子。【Code-verified】

#### 5.3.4 因果引用和共享实体

**因果链接**来自第二组模型抽取的 `causal_relations`，当前新 Retain 输出的规范关系为 `caused_by`。【Code-verified】

教学输入：“用户喝了拿铁，因此出现腹泻。”假设模型抽取如下结果：

```text
facts[0]：用户喝了拿铁。
facts[1]：用户出现腹泻。
```

第二条事实携带的引用可以是：

```json
{
  "target_fact_index": 0,
  "relation_type": "caused_by"
}
```

保存时将抽取数组编号映射到实际事实 ID：

```text
facts[0] -> ID_A
facts[1] -> ID_B

ID_B --caused_by--> ID_A
结果事实             原因事实
```

代码校验允许的关系类型、目标编号范围并排除自身链接。若处理中丢弃某条事实，会调整因果编号映射，避免误指向其他事实。权重固定为 `1.0`，它不是因果已获验证的置信度；因果判断仍来自模型输出。历史导入和既有数据另兼容 `causes`、`enables`、`prevents` 等旧类型。【Code-verified】

**共享实体关联**使用第 9 项解析得到的身份。例如两条事实都通过 `unit_entities` 关联 Alice 的同一个实体 ID，检索可以从一条事实经该 ID 展开到另一条事实。【Code-verified】

```text
Fact A -> Alice 实体 ID <- Fact B
```

这与第 9 项的实体共现分属不同层次：实体共现连接 Alice、Acme 等实体节点；共享实体关联连接提到共同实体的事实。【Code-verified】

当前 SQL 路径没有为所有共享实体事实对预写 `memory_links` 的 Entity 行。检索按共享实体 ID 展开；图接口读取事实实体关联得到 `canonical_name`，按名称分组为可见事实生成 Entity 展示边，并限制数量。【Code-verified】

观察节点还有展示推导：可以继承来源事实的实体和链接；共享来源事实的观察之间还可以生成 Semantic 展示边。因此图中的连线可能是存储的事实链接，也可能是展示时推导的关系，不能把每条可见线都视为 Retain 当时写入的一行链接。【Code-verified】

#### 5.3.5 写入时机和源码证据

普通 SQL 路径大体为：【Code-verified】

```text
写入前：实体解析 + 搜索已有事实的语义邻居
                       |
                       v
写入事务内：
  保存事实与实体关联
  创建时间链接
  创建批内及已预计算的语义链接
  创建因果链接
                       |
                       v
                     提交
```

流式路径可以把语义链接延后到所有小批次提交后统一计算。最终 ANN 是 best-effort，出错会记录警告，已经提交的事实不会整体回滚；普通事务内的链接写入异常会向外传播，影响该事务。如果记忆存储后端自行生成语义链接，流水线跳过重复的最终 ANN。【Code-verified】

路径相对 `hindsight-api-slim/hindsight_api/`：

- `engine/retain/orchestrator.py`：`_pre_resolve_phase1()`、`_insert_facts_and_links()`、`_run_final_semantic_ann()` 及流式调用分支。
- `engine/retain/link_utils.py`：语义 ANN 和批内计算、时间权重和截取、因果编号转换；`engine/retain/link_creation.py` 为相应入口封装。
- `engine/db/ops_postgresql.py`：`fetch_temporal_neighbors()`，当前库内时间邻居查询没有硬性 24 小时限制。
- `engine/memory_engine.py`：图接口的实体展示边、观察继承链接和共享来源展示边；`engine/causal_links.py` 为因果类型定义。
- `config.py`：`DEFAULT_SEMANTIC_LINK_MIN_SIMILARITY = 0.7`。默认值不冒充当前容器的生效配置；以上行为以核对的本地源码为准。

### 5.4 第 12 项 识别可以复用的旧块

> 状态：已共同阅读并确认写入，2026-10-04。涵盖块哈希、按编号比较的四类结果、旧记忆清理、元数据更新、强制重抽及并发保护。

#### 5.4.1 同一文档更新时只重新处理变化部分

Delta Retain 即增量写入。基本场景是同一个 bank 下更新同一个 `document_id`：先比较新旧块，再决定哪些内容需要进入事实抽取流水线。【Code-verified】

```text
旧文档：  A        B        C
          |        |        |
新文档：  A        B'       C
          |        |        |
处理：   复用    重新处理   复用
```

未变化的块保留已有事实、事实 ID、向量和关联；已修改及新增的块重新进行抽取、向量生成、实体解析和链接创建。块复用既减少模型费用，也保留未变事实的身份及其上层知识证据，不属于可以忽略的普通工程细节。【Code-verified】

#### 5.4.2 按同一块编号比较 SHA-256

`compute_chunk_hash()` 的实现为 `sha256(chunk_text.encode()).hexdigest()`。新内容先按当前生效的普通或结构化切块配置切分，再比较相同编号的旧块哈希与新块哈希。【Code-verified】

```text
新内容 -> 切块 -> 每个块计算 SHA-256
                        |
                        v
比较旧 chunk[index] 和新 chunk[index]
                        |
                        v
               分类变化结果
```

| 类别 | 判断 | 处理 |
| --- | --- | --- |
| unchanged 未变化 | 同编号，哈希一致 | 复用已有事实 |
| changed 已修改 | 同编号，哈希不同 | 替换旧块及其事实 |
| new 新增 | 新编号没有旧块 | 抽取并保存新事实 |
| removed 已删除 | 旧编号在新块列表中消失 | 删除旧块及其事实 |

此处没有调用 LLM，也没有用 Embedding 判断语义近似。只要进入哈希的块文本改变，就不满足同编号同哈希的复用条件。上游文本处理和切块决定了实际进入比较的文本。【Code-verified】

例如“ Alice 喜欢 Python。”与“ Alice 很喜欢 Python。”即使意思接近，块文本不同仍会被识别为变化。当前 Delta 比较按编号进行，挪动段落、插入内容或改变切块边界，可能让后续多个块无法复用；不能假设在文档任意位置找到同内容就一定复用。【Code-verified】

#### 5.4.3 修改和删除怎样影响已有记忆

修改 B 时，旧 B 派生的事实及链接需要清理，新 B' 则重新抽取。SQL Delta 写入在事务内更新文档、删除变化或移除的旧块、保存新块并插入新事实及关联。【Code-verified】

```text
旧 B
 +-> 旧事实：删除
 +-> 指向旧事实的链接：清理
 +-> 依赖旧事实的观察：清理受影响的旧观察

新 B'
 +-> 重新抽取、向量、实体、链接
```

旧块删除前会查出其事实 ID，清理受影响的旧观察，避免旧来源消失后过时观察仍可检索。还会登记相关实体清理候选和剩余事实的补链维护候选。这些维护算法属于后续组件，但它们由本次记忆替换触发，是本项必须说明的副作用。【Code-verified】

仅删除块时，即使没有新事实需要抽取，仍要走删除写入，不能当作内容未变。所有块均未变时，可只更新文档及已有记忆的 metadata、tags 等信息，抽取和向量计算不重做。因此复用事实不表示数据库没有任何更新。【Code-verified】

#### 5.4.4 适用条件和运行边界

1. **文档身份**：需要能确定同一 bank 下的一个 `document_id`。换 ID 会作为其他文档处理，不能据此复用上一份文档的块。没有旧块或旧块缺哈希时，Delta 返回回退信号，进入完整写入路径。
2. **切块和无匹配**：新旧切块边界变化会影响命中。通常没有任何未变块时回退完整写入；超大文档严格追加另有保护分支，避免误删原有内容。Delta 的入口还受首个子批次等条件限制。
3. **抽取配置变化**：哈希比较基于块文本。更换模型、Prompt 或抽取规则，不保证相同正文自动重抽；`force_reextract` 会绕过 Delta 和相同内容恢复跳过，重新处理内容。
4. **追加和恢复**：追加基于已存正文与新增内容，需核对所基于的旧版本；比较时不能把新增尾部误当作整份文档。流式恢复可以按已提交的块哈希跳过已完成工作，这与普通按块编号的 Delta 比较是相关但不同的路径。
5. **并发更新保护**：SQL 路径在模型抽取前重新检查文档哈希，写入事务内再锁定文档并核对版本；版本改变时放弃过期 Delta 计划并回退处理。其他存储后端有自己的版本条件写入保护，不能把 SQL 行锁当作所有后端的实现。【Code-verified】

#### 5.4.5 实际位置和源码证据

```text
提交同一文档的新版本
          |
          v
重新切块，比较旧块哈希
          |
   +------+------+
   |             |
未变的块     修改 / 新增的块
   |             |
保留事实     抽取、向量、实体、链接

移除的块 -> 清理旧事实及受影响的关系
```

第 12 项决定哪些内容进入后续流水线，实际可以发生于重新抽取以及第 9、10、11 项之前。清单编号用于共同学习，不能当作严格执行顺序。【Code-verified】

路径相对 `hindsight-api-slim/hindsight_api/`：

- `engine/retain/chunk_storage.py`：`compute_chunk_hash()`、`load_existing_chunks()`、`delete_chunks_by_ids()`，哈希、旧块读取及删除副作用。
- `engine/retain/orchestrator.py`：`_chunk_contents_for_delta()`、`_classify_chunk_diff()`、`_try_delta_retain()`，切块、四类比较结果、变更处理和版本核对。
- 同文件：`_delta_metadata_only()`、`_build_delta_contents()`、`force_reextract` 入口，以及流式恢复的已提交块哈希判断。
- `engine/retain/fact_storage.py`：旧观察清理、metadata 和 tags 更新等辅助操作；其他存储后端的条件写入由对应存储实现负责。

## 6. 源码证据与下一步

路径相对 `hindsight-api-slim/hindsight_api/`：

- `api/http.py`：`api_retain()` 中同步请求与后台任务的分流。
- `engine/memory_engine.py`：`retain_async()`、`retain_batch_async()`、`submit_async_retain()`、`_submit_post_insert_maintenance()`。
- `engine/retain/orchestrator.py`：`retain_batch()`、`_extract_and_embed()`、`_streaming_retain_batch()`。
- `engine/retain/fact_extraction.py`：切块、Prompt、抽取及结果转换。
- `engine/retain/embedding_processing.py`、`entity_processing.py`、`link_creation.py`：向量输入加工、实体解析和链接创建。

当前已写入第一、二组和第三组第 9 至 12 项。2026-10-04 用户反馈：实体解析方式和 Embedding 输入加工方式是本轮明确学到的内容；随后共同补齐第 11 项链接创建、第 12 项增量块复用，并确认写入。

本轮清单核对：

| 分组 | 共同阅读状态 | 写入状态 |
| --- | --- | --- |
| 第一组 第 1 至 4 项 切块 | 已共同阅读 | 已写入 |
| 第二组 第 5 至 8 项 事实抽取 | 已共同阅读 | 已写入 |
| 第三组 第 9 至 12 项 实体、向量输入、链接和增量复用 | 已共同阅读 | 已写入 |

组件 02 的既定 Retain 清单已完成，下一站为组件 03 记忆表示与持久化，尚未开始共同阅读。后续按组件地图继续：检索如何使用这些关系在 04 展开；观察归纳和心智模型刷新在 09 展开；后台维护和任务控制在 10 展开。完成标记限于本轮已核对的清单，后续发现新的 Runtime 操作仍须主动补入并说明。
