# Hindsight 记忆表示与持久化

> 组件 03：第一组「对象关系」、第二组「记录字段」和第三组「存储接口」已共同阅读并确认写入；第四、五组待共同阅读。更新日期：2026-10-08。
>
> 承接 [[02_retain_pipeline]]：02 关注信息怎样被处理成记忆，03 关注处理完成后留下什么、怎样关联和持久化，以及变化时怎样维护。本文按共同阅读进度补充，不能把第一至三组完成当作整个组件完成。第三组按共同确认的人话讲解重新组织，源码名称集中放在定位部分。

## 1. 本组件要回答什么

对应 [[memory_抽象层|Memory 抽象层]]，本组件主要回答：信息要留成什么形态，才能以后找到、追溯和修改？【Inferred：组织理解的抽象】

以下对象关系以当前本地源码的默认 PostgreSQL 实现为范围。表名表示逻辑记录及职责，不代表必须部署多个数据库；替换存储的接口和所有权边界见第三组。

## 2. 阅读清单

| 分组 | 核心问题与操作范围 | 共同阅读状态 | 写入状态 |
| --- | --- | --- | --- |
| 第一组：对象关系 | Bank；Document / Chunk 来源；Memory Unit；实体及事实关联；共现统计；显式事实链接；Observation 证据引用；Mental Model 对象边界 | 已共同阅读 | 已写入 |
| 第二组：记录字段 | 身份、正文与类型；上下文、标签和 metadata；事件时间、提及时间和记录时间；向量与关键词信号；来源、附件、归纳范围、证据数及处理状态 | 已共同阅读 | 已写入 |
| 第三组：存储接口 | 存储负责哪些动作、读写交接形态、默认 SQL 路径、替换存储及责任接管、原文保存与模型可见内容 | 已共同阅读 | 已写入 |
| 第四组：写入一致性 | 事实、向量、实体关联和链接的写入边界；分批提交、约束和引用；版本冲突、写入冻结及失败恢复的行为影响 | 待共同阅读 | 未写入 |
| 第五组：更新与删除 | 手工修改与重建向量；失效、归档和恢复；文档或片段删除牵连；来源变化后的观察清理；实体清理、重建关系、历史和变化时间 | 待共同阅读 | 未写入 |

这五组是当前研究清单；后续发现影响 Runtime 的新操作，须补入并主动说明。归纳决策和心智模型刷新机制仍按组件地图在 09 展开，此处研究其保存形态及依赖关系。

## 3. 第一组：对象关系

### 3.1 一份输入最后留下哪些对象

教学输入：

~~~text
Alice 在 Acme 公司使用 Python 编程。
Alice 喜欢喝黑咖啡。
~~~

假设抽取出了两条事实，关系可理解为：

~~~text
Bank：某一个记忆库
|
+-- Document D1：整份输入及文档信息
|      |
|      +-- Chunk C0：切出来的原文片段
|             |
|             +-- Fact F1：Alice 在 Acme 使用 Python 编程
|             +-- Fact F2：Alice 喜欢喝黑咖啡
|
+-- Entity：Alice、Acme、Python……
|      ^
|      | unit_entities：哪条事实提到了哪个实体
|      +-- F1 -> Alice、Acme、Python
|      +-- F2 -> Alice……
|
+-- memory_links：保存满足规则的显式事实关系
|                  例如 Semantic / Temporal / Causal
|
+-- Observation O1：后续 Consolidation 生成的归纳
       +-- source_memory_ids -> 支撑它的事实 ID
~~~

D1、C0、F1 等是教学代号。具体生成哪些事实、实体、观察或链接取决于输入和处理规则；不是每份输入必然生成图中全部对象。【Inferred：教学示例；对象关系 Code-verified】

### 3.2 Document / Chunk：来源

Document 保存整份输入的身份及文档信息；Chunk 表示其中一个片段。事实可用 document_id 和 chunk_id 指向来源：

~~~text
F1
+-- document_id -> D1
+-- chunk_id    -> C0
~~~

这些关联支持回到对应文档或片段，回答“这条记忆来自哪里”。来源字段允许为空，不能把所有 Memory Unit 都理解为必须对应一个原文片段；Observation 尤其通过源事实列表表达证据。【Code-verified】

默认配置保留原文，但原文保存可以关闭。因此，存在来源 ID 不保证能读取原文正文。替换存储还可能在 SQL 以外保存原文，具体边界留在第三组。【Code-verified】

### 3.3 Memory Unit：记忆内容

界面中的世界事实 world、经历 experience 和观察 observation，在默认实现中均可以是 memory_units 的记录，通过 fact_type 区分。Fact 是其中的内容角色，不意味着另有一个名为 facts 的表。【Code-verified】

~~~text
memory_units
+-- F1：fact_type = world
+-- F2：fact_type = world
+-- O1：fact_type = observation
~~~

这些记录承载可被找回的记忆内容，但形成方式及来源不同。默认实现中，正文和 embedding 保存于同一条记忆记录，向量类型由 pgvector 支持，不要求单独部署向量数据库。【Code-verified】

### 3.4 Entity 与 unit_entities：稳定身份及提及关系

Alice 有独立实体记录，携带实体 ID 和 canonical_name。事实与实体通过 unit_entities 关联：

~~~text
事实 ID     实体 ID
-----------------
F1          EA
F2          EA
~~~

这表示两条事实都提到了同一个 Alice。一条事实可以提到多个实体，一个实体也可以出现在多条事实中；不需要把 Alice 的全部经历放进一个不断增大的对象。【Code-verified + Inferred：设计解释】

02 的实体解析最终形成稳定实体身份，以及事实到实体的关联。实体名称是可读的表示，关联使用的是实体 ID。【Code-verified】

### 3.5 三类关系记录不能混为一谈

| 记录 | 关联对象 | 作用 |
| --- | --- | --- |
| unit_entities | 事实与实体 | 哪条事实提到了谁 |
| entity_cooccurrences | 实体与实体 | 实体共同出现的计数和最近共现时间 |
| memory_links | 记忆与记忆 | 显式语义、时间、因果等链接 |

共享实体是事实之间的一种关系，但当前默认 SQL 路径不为所有共享实体事实对预写 Entity 类型的 memory_links 行。F1 -> Alice <- F2 已表达了可沿共享实体展开的关系。图接口还会利用事实实体关联生成展示边，因此 UI 中的一条线不一定对应数据库中独立保存的一条边。【Code-verified】

各类关系如何判定详见 [[02_retain_pipeline]] 第 11 项；本组关注它们保存在哪里和关联什么。

### 3.6 Observation：归纳内容与证据分别保存

承接共同体验过的 Alice / Alice Chen 例子：Consolidation 可以更新原有 Observation 正文，并追加支持它的新事实引用，而不是必须改写原始 Fact。

~~~text
F1：Alice 在 Acme 使用 Python
F2：Alice Chen 在 Acme 使用 Python
             |
             v
更新 Observation O1
+-- text：Alice Chen 在 Acme 使用 Python
+-- source_memory_ids：[F1, F2]
~~~

默认 PostgreSQL 用 memory_units.source_memory_ids 保存源事实 ID 列表。归纳后的内容和支撑它的证据因此可以分别维护；来源变化后的检查和清理在第五组展开，模型怎样决定合并在组件 09 展开。【Code-verified】

共同确认的关系：事实经过归纳，可以生成新观察或更新已有观察；原始事实通常仍保留，观察是额外的归纳层。一个观察可以关联多个来源事实，同一事实也可以支持多个观察。Consolidation 根据新事实和召回的相关旧观察，决定创建、更新、删除观察，或者不修改；对观察做合并，不会自动把来源事实合并或删除。【Code-verified】

Mental Model 是另一类单独保存于 mental_models 的对象，不能等同于 Observation，也不能认为每条 Observation 必然生成一条 Mental Model。【Code-verified】

### 3.7 第一组的抽象

~~~text
原文和片段：提供来源
事实和观察：承载记忆内容
实体：提供稳定身份
向量和关系：帮助以后找到内容
ID 引用：把这些对象关联起来
~~~

这是从已核对实现提炼的职责分工，不要求所有 Memory 系统都有这些对象或部署相同基础设施。【Inferred】

## 4. 第二组：记录字段

### 4.1 一条记忆携带的五类信息

教学输入为「Alice 在 2025 年 3 月 1 日加入了 Acme 公司」。一条事实的逻辑信息可理解为：

~~~text
这条记忆
|
+-- 身份和内容：ID、所属 Bank、正文、记忆类型
+-- 时间：事情发生、被提到、首次写入和最近修改的时间
+-- 上下文和分类：context、tags、metadata、归纳范围
+-- 检索信息：embedding、关键词信号、实体关联
+-- 来源和处理状态：文档、片段、附件、支撑事实、归纳状态
~~~

这不是模型一次输出的 JSON，也不是所有字段都存于同一张表的结构图。字段和关联分别来自模型抽取、调用者输入及系统生成；本文描述逻辑职责和默认 SQL 实现。【Code-verified + Inferred：教学组织】

| 字段 | 人话 |
| --- | --- |
| id；读写模型中的 unit_id | 这条记忆的稳定身份 |
| bank_id | 所属记忆库 |
| text | 系统记住的内容 |
| fact_type | 内容是世界事实、经历还是观察 |

身份和正文分开，使一条记录更新正文后仍可以被原来的 ID 引用。新事实并不因此必然覆盖旧事实；创建或更新的决策属于归纳及更新逻辑。【Code-verified + Inferred：设计解释】

### 4.2 时间：事件、提及和数据库记录分别保存

教学例子中可以同时存在：事情发生于 2025-03-01，对话发生于 2026-09-30，直到 2026-10-05 才导入系统。这几个日期回答不同问题，不应合成一个时间。【Inferred：教学示例】

| 字段 | 含义与默认事实写入行为 |
| --- | --- |
| occurred_start / occurred_end | 事件发生的开始和结束时间；未知时可以为空 |
| mentioned_at | 来源对话或文档中提到信息的时间；普通 Retain 抽取结果继承输入的 event_date，不是数据库导入时间 |
| event_date | 默认普通事实写入优先使用 occurred_start，缺失时使用 mentioned_at；两者都为空时仍为空 |
| created_at | 这条记忆记录首次创建的时间 |
| updated_at | 记录最近写入或更新的时间；可用于检测变化 |

~~~text
Alice 去年发生了什么？      -> 事件时间
这件事是哪次对话提到的？  -> mentioned_at
昨天新增了哪些记录？      -> created_at
哪些记录最近变化了？      -> updated_at
~~~

这些箭头解释时间字段的职责，不保证 API 的同名筛选参数一定按直觉映射；具体 Recall 时间过滤仍按组件 04 核对。【Inferred：职责示例】

导入旧对话时，可以传旧对话的时间；不能把 mentioned_at 一律解释成系统当前时间。occurred_start / occurred_end 为 null 表示不知道具体事件时间，不要求模型编造日期。【Code-verified】

原始事实被标记已归纳时，会写 consolidated_at，但这个处理记账操作刻意不推进该事实的 updated_at。真正重写 Observation 正文时，会推进 Observation 的 updated_at。事件发生时间、内容修改时间和后台处理时间因此保持区别。【Code-verified】

### 4.3 上下文、标签和额外属性

| 字段 | 职责 |
| --- | --- |
| context | 附带场景，例如「员工入职访谈」 |
| tags | 筛选和组织记忆的标签 |
| metadata | 额外结构化属性，例如来源系统、会话编号 |
| observation_scopes | 指定原始事实参与哪些范围的观察归纳 |

~~~text
text：Alice 加入了 Acme。
context：员工入职访谈
tags：[project:acme, session:42]
metadata：{source: chat, conversation_id: 42}
~~~

普通 Retain 抽取路径把调用者输入的 context、metadata、tags 带到事实中；启用了 tag 输出的分类维度也可以生成标签。具体分类与自由实体的区别已在 [[02_retain_pipeline]] 第二组说明。【Code-verified】

### 4.4 检索信息：向量和关键词是不同线索

~~~text
事实正文及补充信息
       |
       +-- embedding：为语义检索准备的向量
       +-- text_signals：为关键词检索准备的文字信号
       +-- 实体关联：沿稳定实体身份找相关事实
~~~

Embedding 输入中的日期、实体加工已在 02 第 10 项讨论。默认事实写入另行构造 text_signals，包含实体名称及 occurred_start / occurred_end 的日期文字。例如可出现「Alice Acme March 1 2025」，用于补充关键词检索。【Code-verified；示例 Inferred】

底层按选用的全文检索实现建立索引：某些实现用 search_vector 字段，另一些直接索引正文等列。不能把所有后端理解成同一种物理索引形态；这些检索路径如何召回、融合和裁剪留到组件 04。【Code-verified】

### 4.5 来源、证据和处理状态

| 来源字段 | 含义 |
| --- | --- |
| document_id / chunk_id | 来源文档和片段 |
| attachment_ids | 这条事实来自哪些附件；是按事实归属保存，不等于整个文档的全部附件 |
| source_memory_ids | 支持 Observation 的来源事实列表 |

| 状态字段 | 含义 |
| --- | --- |
| proof_count | 支持观察的来源事实数量；不是置信度百分比，也不保证每条证据来自独立来源 |
| consolidated_at | 原始事实已被归纳流程处理的时间；不保证新建了一条观察 |
| consolidation_failed_at | 归纳处理失败的标记时间 |
| edited_at | 手工编辑路径修改的时间 |

归纳可能新增观察、更新旧观察或没有新增内容，因此「已归纳」不等于「生成新观察」。这些字段支持后台区分待处理、已处理和失败状态；具体重试、重处理及失效清理在第三至五组及组件 09、10 按所属职责继续展开。【Code-verified】

### 4.6 observation_scopes：决定整理范围，而不是合并算法

共同澄清的三个概念：

| 名称 | 职责 |
| --- | --- |
| tags | 原始事实的标签 |
| Retain 输入中的 observation_scopes | 这条事实将按哪些标签范围参与归纳 |
| Consolidation | 召回相关旧观察，让模型决定创建、更新、删除或不修改，再检查并保存结果 |

combined 是范围选择模式，不是 Consolidation 操作的另一个名字。「观察范围」在这里主要指本轮用于召回旧观察的标签条件，以及新建观察采用的标签范围。【Code-verified】

对标签为 [project:acme, session:42] 的事实，各模式解析为：

| 模式 | 归纳轮次和标签范围 |
| --- | --- |
| combined；默认 | 一轮：[project:acme, session:42] |
| per_tag | 两轮：[project:acme]、[session:42] |
| all_combinations | 三轮：[project:acme]、[session:42]、[project:acme, session:42] |
| shared | 一轮：空范围 [] |
| 显式 list[list[str]] | 按列出的每组标签分别执行 |

同一事实可以在多个范围参与整理；每轮分别召回和调用模型，并不保证每轮必然新增观察。原始事实仍可以保留原来的完整标签。【Code-verified】

非空范围的候选召回使用 all_strict：旧观察必须包含全部请求标签，可以有额外标签，排除不带标签的观察；这不是标签集合完全相等。shared 解析为空范围，当前归纳候选召回使用 any 且不做标签过滤，仍限制在当前 Bank；新建的共享观察不带标签，但召回不只限于空标签旧观察。【Code-verified】

#### 两次会话为何可能需要单独指定项目范围

~~~text
F1：Alice 在 Acme 使用 Python。
tags：[project:acme, session:42]

F2：Alice Chen 在 Acme 使用 Python。
tags：[project:acme, session:43]
~~~

两次都采用 combined 时，F2 的候选要求包含 session:43；如果 F1 生成的旧观察只有 project:acme 和 session:42，就不能被该标签条件召回，即使内容很相似。【Code-verified + Inferred：教学例子】

要让不同会话按项目持续整理，可在两次 Retain 时都显式选择项目范围：

~~~json
{
  "content": "Alice Chen 在 Acme 使用 Python。",
  "tags": ["project:acme", "session:43"],
  "observation_scopes": [["project:acme"]]
}
~~~

原始事实仍保留项目和会话标签；本轮归纳使用项目标签，从相关候选中判断是否更新已有观察。显式 scopes 是列出归纳使用的标签组合，不是给原始事实追加 tags；范围相同只提供比较机会，不保证合并。【Code-verified + Inferred：教学解释】

#### 归纳确实先检索候选，再让模型比较

~~~text
取得尚未归纳的原始事实
           |
解析该事实的 observation_scopes
           |
用每条新事实正文作为 query
召回当前 Bank 内的相关旧 Observation
  +-- 按本轮标签条件筛选
  +-- 按检索预算和 token 预算限制结果
  +-- 携带相应来源事实及日期
           |
同批候选观察按 ID 去重
           |
新事实 + 相关旧观察及证据 -> 模型
           |
creates / updates / deletes
           |
检查操作对象及来源 -> 应用修改
~~~

标签条件决定候选资格，相关性检索选择候选，模型决定整理动作。当前源码的主路径以新事实正文查询旧 Observation，而不是把整个 Bank 全量送给模型。默认归纳 Recall 预算为 low、观察结果 token 上限为 512，配置可以覆盖；归纳候选采用 interleave 路径。这里只确认共同讨论的主流程，不表示组件 09 的全部归纳机制已阅读完成。【Code-verified】

#### Retain 与手动触发请求中的同名字段职责不同

| 设置位置 | 决定什么 |
| --- | --- |
| Retain 输入上的 observation_scopes | 这条事实以后按哪些范围参与归纳 |
| 手动触发 Consolidation 请求上的 observation_scopes | 本次任务挑哪些尚未归纳的原始事实来处理 |

例如，触发请求指定 [["project:acme"]] 时，只选标签包含 project:acme 的待处理事实。选出来之后，仍按每条事实在 Retain 时保存的范围规则执行；任务筛选不会自动重写这些事实的归纳范围。省略该任务筛选时处理全部符合待处理状态的事实，仍受运行预算和分批机制约束。【Code-verified】

### 4.7 第二组的抽象和共同确认

一条记忆是一段内容，带着身份、时间、适用场景、找回线索，以及来源和处理状态。【Inferred：通用抽象】

~~~text
模型：抽取内容、事件时间、实体等
调用者：提供场景、来源时间、标签、metadata 等
系统：分配 ID、生成向量、维护引用和处理状态
~~~

以上是普通抽取路径的职责说明，不要求所有模式都调用模型，chunks 等模式的差异已在 02 记录。事实与观察关系已补入 3.6：观察是归纳层，可以绑定多条事实；同一事实也可支持多个观察；归纳增删改主要作用于观察，不会因合并观察自动合并或删除原始事实。【Code-verified】

## 5. 第三组：谁负责把记忆存下来

### 5.1 先看一次交接

教学输入仍是「Alice 在 Acme 公司使用 Python」。Retain 已经准备好事实正文、向量和实体信息，接下来需要把它们保存，支持以后找回和修改。

~~~text
记忆处理部分                         存储部分
这些事实和来源准备好了 -------------> 保存
找 Alice 的相关记忆 ----------------> 查询并返回
按 ID 读这一条记忆 -----------------> 返回对应记录
这条记忆需要修改或删除 -------------> 维护记录及关联
~~~

两边通过共同约定交接。上层表达要保存、读取或维护什么，下层实现完成这些动作并返回约定结果。不同存储实现可以复用同一套交接方式。【Code-verified + Inferred：教学解释】

| 存储能力 | 人话 |
| --- | --- |
| 写入 | 把处理后的事实及关联保存下来 |
| 按身份读取和浏览 | 给一个 ID 读对应记录，或分页浏览记忆 |
| 检索 | 按问题查找候选内容 |
| 修改与维护 | 支持编辑、删除、归档、恢复及关系维护 |
| 后台状态 | 找待归纳事实、标记处理结果、检查记忆是否变化 |

因此，只实现向量搜索不等于承担完整记忆存储职责。替换实现需要支持其接管范围内的读写、检索和维护约定。【Code-verified + Inferred：设计解释】

### 5.2 写入、按 ID 读取和按问题检索的交接不同

~~~text
写入：
正文 + 向量 + 类型 + 时间 + 来源及关联 -> 存储

按 ID 读取：
读取 F1 -> F1 的正文、类型、时间、标签、来源等

按问题检索：
Alice 用什么编程语言？ -> 候选记忆及本次检索分数
~~~

完整写入记录可包含实体和因果引用；默认 SQL 中的部分关联则拆在独立表中。按 ID 读取没有本次查询的排序分数，检索候选才带有相关性信息。记忆本身的属性与它对某次问题的相关性不是同一件事。【Code-verified】

### 5.3 默认情况下保存到数据库

默认路径在 PostgreSQL 中保存来源原文和片段、事实和观察及其向量、实体与关联、事实关系。教学例子中分别保存 F1 正文、F1 提到 Alice / Acme / Python 的关联，以及 F1 来源 D1 / C0 的引用，供后续读取和检索。【Code-verified；教学代号 Inferred】

默认路径不要求为向量和图关系另外部署两个数据库。数据库层另有 PostgreSQL 和 Oracle 适配；「适配 SQL 方言和连接」与「替换整套记忆存储」是不同职责。【Code-verified】

默认 SQL 保存事实时同时维护相应数据库索引。某些扩展补充入口在默认实现中无需再做一次写入，因为数据已经由 SQL 路径保存；不能据此认为没有建立索引。语义关系后续补建与事务范围放在第四组讲。【Code-verified】

### 5.4 换存储可能同时改变责任分工

最简单的变化是把原先交给 PostgreSQL 的记忆交给另一个服务。接口也允许该服务接管更多工作：处理实体名称、保存实体关联、建立索引、保存来源文档及片段，再回答查询。换存储因此可能改变实体匹配、关系维护、提交方式和检索行为，不能只把它看成换一个数据保存地址。【Code-verified + Inferred：设计解释】

| 边界 | 默认路径 | 自定义存储可以承担的职责 |
| --- | --- | --- |
| 保存和实体处理 | 引擎组织 SQL 写入，默认路径处理实体身份 | 接管记忆和来源保存，由自己的实现处理实体身份 |
| 分批提交 | 引擎沿默认写入路径提交 | 接收陆续产生的批次，自行决定缓存和提交时机 |
| 检索 | 存储返回多路候选，引擎继续融合、排序和裁剪 | 可接管完整检索，包括预算、排序和来源补充 |
| 来源和索引 | 从 SQL 记录及索引读取 | 可提供独立文档读取、内部语义关系维护和搜索索引 |

完整检索是可选接管：存储支持某次请求时返回完整结果，不支持时返回不接管信号，交回引擎执行原有流程。两种路径应遵守同样的筛选、评分、结果长度和来源要求，不能随意近似不支持的请求。【Code-verified】

分批接收不意味着整个文档一次提交，中止也不保证撤销已经提交的数据；这是第四组的一致性问题。是否接管保存按 Bank 判断，接口允许一个扩展对不同 Bank 有不同分工。【Code-verified】

接管记忆存储不等于整个项目脱离 SQL。例如知识页的权威记录仍在 SQL 中，扩展可保存和维护它的派生检索索引。恢复该索引时应以权威记录为依据，不能把索引副本当成原始记录。【Code-verified】

扩展须实现交接约定；仅填新地址不会自动适配任意存储。系统启动时加载、初始化存储实现，关闭时释放其资源。【Code-verified】

### 5.5 保存事实不等于保存完整原文

~~~text
来源：一整段关于 Alice 工作经历的对话
事实：Alice 在 Acme 使用 Python。
~~~

默认 SQL 实现可以保留原文，也可以关闭原文保存。开启时能从事实回看来源片段；关闭时文档原文为空、片段正文为空，事实和向量仍可保存检索，来源身份及内容 hash 仍可以保留。【Code-verified】

因此系统可能能找回提炼后的事实，却无法展开完整来源语境。这改变模型能获得的证据，属于 Runtime 研究范围。【Inferred：由存储行为推导】

自定义存储的原文与片段要按实际实现核对。已检查的扩展文档写入路径分别传整篇原文和 chunk 正文，关闭整篇保存时该路径仍会传 chunk 文本；不能把默认 SQL 的开关效果直接套到全部扩展。【Code-verified】

### 5.6 第三组的抽象及源码定位

上层决定要记住什么、要找什么；存储实现负责保存、查询和维护，并明确哪些责任由它接管。它可以提供底层保存与候选查询，也可以接管实体匹配、提交方式和完整检索。【Inferred：通用抽象】

在建立以上行为理解后，源码名称对应如下：

| 已解释的职责 | 源码名称 |
| --- | --- |
| 记忆存储和检索的共同约定 | MemoriesExtension |
| 默认 SQL 记忆存储 | PostgresMemories；数据库适配另由 DataAccessOps / DatabaseBackend 完成 |
| 完整记录、按 ID 读取、检索候选的不同形态 | FactRecord、StoredMemory、RecallArms；Retain 前段还使用 ProcessedFact |
| 按 Bank 判断接管范围、交接批次、可选完整检索 | store_owned_for、begin_retain / RetainSession、full_recall |
| 扩展加载与原文保存设置 | HINDSIGHT_API_MEMORIES_EXTENSION；store_document_text / HINDSIGHT_API_STORE_DOCUMENT_TEXT |

## 6. 源码锚点

路径相对 hindsight-api-slim/hindsight_api/：

- engine/memories/base.py、engine/memories/__init__.py、extensions/loader.py：读写模型、存储责任边界、批次会话、完整 Recall 和扩展加载。
- engine/memories/postgres.py、engine/memories/pg/writes.py、engine/memories/pg/reads.py：默认存储、事实字段、手工修改和归纳状态标记。
- engine/retain/chunk_storage.py、engine/chunk_ids.py、engine/retain/orchestrator.py、engine/db/、config.py：来源片段、身份构造、默认与扩展写入分流、SQL 适配及原文保存配置。
- engine/retain/link_creation.py、engine/retain/link_utils.py、engine/entity_resolver.py：事实关联与实体共现处理；详见 02。
- engine/consolidation/consolidator.py、engine/search/tags.py、api/http.py、alembic/versions/：归纳范围解析、候选召回、标签匹配、触发筛选、Observation 来源引用及 mental_models 独立存储形态。

## 7. 当前阅读位置

第一至三组已共同阅读并写入，包含 observation_scopes、事实与观察的问答及第三组的人话重讲。第四、五组待共同阅读；下一步是第四组写入一致性。组件 03 尚未整体完成，组件 09 的归纳执行机制也未因本轮说明而标记完成。
