# 模型到底看见了什么？AI 应用开发绕不开的 Chat Template

## 开场：我真正想问的，不是 Transformer

我面试过不少想做 AI 应用开发的人。经常会问一个问题：**“你了解大模型的底层原理吗？”**

多数回答大同小异：Transformer、注意力机制、词表、把文本切成 Token，然后预测下一个 Token……这些答案当然没有错。但听到这里，我通常还会继续追问：

> 当你在聊天框里输入一句话，连同系统提示词、历史消息、工具定义一起发给模型时，模型最终“看到”的明文，到底是什么样子？

这个问题往往会让对话停下来。

我并不期待每一个 AI 应用开发者都能推导 Transformer 的公式，更不认为那是做好 Agent 的前提。但如果你要把大模型接入真实产品，设计多轮对话、接入工具、做 RAG，甚至只是从 Qwen 换到 Llama，你至少需要理解一件事：**你在 API 里传入的 `messages`，并不是模型真正的输入。**

在模型的世界里，没有 `system`、`user`、`assistant` 这些天然存在的“角色”，也没有一条条结构化的聊天消息。它最终接收到的，只是一段按照特定规则拼接好的文本，再被切分成 Token。这套把聊天消息翻译成模型输入的规则，就叫作 **Chat Template**。

这篇文章不讲 Transformer 的数学推导。我们就从一份真实的 Qwen Chat Template 出发，看清楚：一次看似普通的聊天请求，究竟是怎样一步步变成模型输入的。

## 先看开发者写了什么

以 LangChain 为例。你写 Agent 时，通常操作的是几种消息对象：`SystemMessage`、`HumanMessage`、`AIMessage` 和 `ToolMessage`。

```python
messages = [
    SystemMessage("你是一个简洁的旅行助手。"),
    HumanMessage("帮我安排一趟北京两日游。"),
]
```

这段结构清楚、好用，也方便框架管理历史记录。但它不是语言模型直接理解的格式。模型不会读取 Python 对象，更不会天然知道谁是“人类”、谁是“助手”。在请求模型前，框架或模型服务需要先把它渲染成一段文本；这一步，正是 Chat Template 的工作。

## 以Qwen3.5中为例 实际收到的内容

上面两条消息，经过 Qwen 这类使用 `<|im_start|>`、`<|im_end|>` 标记的模板后，大致会变成下面这样：

```text
<|im_start|>system
你是一个简洁的旅行助手。<|im_end|>
<|im_start|>user
帮我安排一趟北京两日游。<|im_end|>
<|im_start|>assistant
<think>
```

最后的 `assistant` 标记不是历史消息，而是在告诉模型：**现在轮到你继续写了。** 模型会从这里开始预测下一个 Token，生成回答；生成结束后，回答会被包装回 `AIMessage`，作为下一轮对话的历史。

这就是容易被 API 遮住的事实：`system`、`user`、`assistant` 并不是模型的内置概念，而是由特殊标记和模板约定出来的角色。换一个模型，这些标记可能完全不同；有的用 `[INST]`，有的用 `<|user|>`，有的还会有专门的思考、工具调用标记。

> `messages` 是应用层展示给我们看的；Chat Template 才是模型具体看到的内容。

## 一份模板，究竟在做什么

你找到的 Qwen 模板虽然很长，但拆开看，核心只做了几件事：

1. **遍历消息**：根据 `system`、`user`、`assistant`、`tool` 等不同角色，写入不同的起止标记。
2. **拼接内容**：把纯文本，或文本、图片、视频组成的内容列表，写到对应消息中。
3. **处理工具**：把可用工具的 JSON Schema 放进系统消息；把模型发起的调用和工具返回结果编码为约定格式。
4. **补上生成起点**：当 `add_generation_prompt=True` 时，在结尾添加 `<|im_start|>assistant`，让模型开始回答。

Jinja 语法里的 `if`、`for`不是什么模型能力，只是模板逻辑：它根据传入的消息，拼出模型训练时见过的那种文本格式。

## 工具调用也只是“写成特定文本”

当 Agent 有工具可用时，Qwen 的模板会把工具定义序列化后放入 system 段。例如你的模板会加入 `# Tools`、`<tools>`，以及每个工具的 JSON 描述，并明确要求模型用 XML 结构输出调用：

```text
<tool_call>
<function=get_weather>
<parameter=city>
北京
</parameter>
</function>
</tool_call>
```

模型并没有真的“调用函数”。它仍然只是在预测下一个 Token，恰好生成了一段符合约定的文本。LangChain 或你的 Agent 程序识别这段文本，执行 `get_weather`，再把结果包装成 `ToolMessage` 传回去。

在这份 Qwen 模板中，工具结果会被放在 `<tool_response>...</tool_response>` 中，并作为下一轮输入的一部分。于是模型可以继续基于工具结果写答案。

这也解释了为什么 Agent 的工具调用偶尔会格式错误：这不是协议天然失效，而是模型没有稳定地生成符合模板约定的文本。

## 图片如何进入对话

图片也是同一个思路，只是内容不再只有文字。在 LangChain 中，一条 `HumanMessage` 可以同时带文字和图片，例如：

```python
HumanMessage(content=[
    {"type": "text", "text": "这张图片里有什么？"},
    {"type": "image_url", "image_url": {"url": "https://example.com/cat.jpg"}},
])
```

Qwen 的模板遇到图片时，不会把图片 URL 原样交给语言模型，而是在文本序列中的相应位置插入类似下面的占位标记：

```text
这张图片里有什么？<|vision_start|><|image_pad|><|vision_end|>
```

随后，多模态处理器会处理实际图片，把它转换成模型可接收的视觉表示，并填到这个位置。你不必为了做应用开发去掌握图像如何切块、如何编码；需要知道的是：**图片不是附在 Prompt 旁边的“附件”，它也会以有位置、有顺序的方式进入模型上下文。**

因此，图片前后的文字、图片的数量，以及文本和图片的排列顺序，都可能影响模型的理解。

## 结语

大模型当然是 Transformer，也确实是在预测下一个 Token。但对 AI 应用开发者来说，更有用的问题是：**它正在预测哪一段文本里的下一个 Token？**

理解 Chat Template，就是从“会调用模型 API”，跨到“知道模型实际上收到了什么”的第一步。

这里是「抽象运行时」的第一篇文章。之后我也会带来更多有关 AI / Agent 开发的内容，欢迎大家在评论区留言讨论。希望这篇文章能帮助你更好地理解 LLM 是如何工作的。


## 最后最后最后，看看它原本的样子

前面提到的规则，落到代码里，就是下面这样一份 Qwen 3.5 Chat Template。它看起来很长，但本质上仍是在做我们刚才拆开的几件事：区分角色、拼接内容、处理图片和工具，并在最后为 assistant 留出生成位置。

```jinja
{%- set image_count = namespace(value=0) %}
{%- set video_count = namespace(value=0) %}
{%- macro render_content(content, do_vision_count, is_system_content=false) %}
    {%- if content is string %}
        {{- content }}
    {%- elif content is iterable and content is not mapping %}
        {%- for item in content %}
            {%- if 'image' in item or 'image_url' in item or item.type == 'image' %}
                {%- if is_system_content %}
                    {{- raise_exception('System message cannot contain images.') }}
                {%- endif %}
                {%- if do_vision_count %}
                    {%- set image_count.value = image_count.value + 1 %}
                {%- endif %}
                {%- if add_vision_id %}
                    {{- 'Picture ' ~ image_count.value ~ ': ' }}
                {%- endif %}
                {{- '<|vision_start|><|image_pad|><|vision_end|>' }}
            {%- elif 'video' in item or item.type == 'video' %}
                {%- if is_system_content %}
                    {{- raise_exception('System message cannot contain videos.') }}
                {%- endif %}
                {%- if do_vision_count %}
                    {%- set video_count.value = video_count.value + 1 %}
                {%- endif %}
                {%- if add_vision_id %}
                    {{- 'Video ' ~ video_count.value ~ ': ' }}
                {%- endif %}
                {{- '<|vision_start|><|video_pad|><|vision_end|>' }}
            {%- elif 'text' in item %}
                {{- item.text }}
            {%- else %}
                {{- raise_exception('Unexpected item type in content.') }}
            {%- endif %}
        {%- endfor %}
    {%- elif content is none or content is undefined %}
        {{- '' }}
    {%- else %}
        {{- raise_exception('Unexpected content type.') }}
    {%- endif %}
{%- endmacro %}
{%- if not messages %}
    {{- raise_exception('No messages provided.') }}
{%- endif %}
{%- if tools and tools is iterable and tools is not mapping %}
    {{- '<|im_start|>system\n' }}
    {{- "# Tools\n\nYou have access to the following functions:\n\n<tools>" }}
    {%- for tool in tools %}
        {{- "\n" }}
        {{- tool | tojson }}
    {%- endfor %}
    {{- "\n</tools>" }}
    {{- '\n\nIf you choose to call a function ONLY reply in the following format with NO suffix:\n\n<tool_call>\n<function=example_function_name>\n<parameter=example_parameter_1>\nvalue_1\n</parameter>\n<parameter=example_parameter_2>\nThis is the value for the second parameter\nthat can span\nmultiple lines\n</parameter>\n</function>\n</tool_call>\n\n<IMPORTANT>\nReminder:\n- Function calls MUST follow the specified format: an inner <function=...></function> block must be nested within <tool_call></tool_call> XML tags\n- Required parameters MUST be specified\n- You may provide optional reasoning for your function call in natural language BEFORE the function call, but NOT after\n- If there is no function call available, answer the question like normal with your current knowledge and do not tell the user about function calls\n</IMPORTANT>' }}
    {%- if messages[0].role == 'system' %}
        {%- set content = render_content(messages[0].content, false, true)|trim %}
        {%- if content %}
            {{- '\n\n' + content }}
        {%- endif %}
    {%- endif %}
    {{- '<|im_end|>\n' }}
{%- else %}
    {%- if messages[0].role == 'system' %}
        {%- set content = render_content(messages[0].content, false, true)|trim %}
        {{- '<|im_start|>system\n' + content + '<|im_end|>\n' }}
    {%- endif %}
{%- endif %}
{%- set ns = namespace(multi_step_tool=true, last_query_index=messages|length - 1) %}
{%- for message in messages[::-1] %}
    {%- set index = (messages|length - 1) - loop.index0 %}
    {%- if ns.multi_step_tool and message.role == "user" %}
        {%- set content = render_content(message.content, false)|trim %}
        {%- if not(content.startswith('<tool_response>') and content.endswith('</tool_response>')) %}
            {%- set ns.multi_step_tool = false %}
            {%- set ns.last_query_index = index %}
        {%- endif %}
    {%- endif %}
{%- endfor %}
{%- if ns.multi_step_tool %}
    {{- raise_exception('No user query found in messages.') }}
{%- endif %}
{%- for message in messages %}
    {%- set content = render_content(message.content, true)|trim %}
    {%- if message.role == "system" %}
        {%- if not loop.first %}
            {{- raise_exception('System message must be at the beginning.') }}
        {%- endif %}
    {%- elif message.role == "user" %}
        {{- '<|im_start|>' + message.role + '\n' + content + '<|im_end|>' + '\n' }}
    {%- elif message.role == "assistant" %}
        {%- set reasoning_content = '' %}
        {%- if message.reasoning_content is string %}
            {%- set reasoning_content = message.reasoning_content %}
        {%- else %}
            {%- if '</think>' in content %}
                {%- set reasoning_content = content.split('</think>')[0].rstrip('\n').split('<think>')[-1].lstrip('\n') %}
                {%- set content = content.split('</think>')[-1].lstrip('\n') %}
            {%- endif %}
        {%- endif %}
        {%- set reasoning_content = reasoning_content|trim %}
        {%- if (preserve_thinking is defined and preserve_thinking is true) or (loop.index0 > ns.last_query_index) %}
            {{- '<|im_start|>' + message.role + '\n<think>\n' + reasoning_content + '\n</think>\n\n' + content }}
        {%- else %}
            {{- '<|im_start|>' + message.role + '\n' + content }}
        {%- endif %}
        {%- if message.tool_calls and message.tool_calls is iterable and message.tool_calls is not mapping %}
            {%- for tool_call in message.tool_calls %}
                {%- if tool_call.function is defined %}
                    {%- set tool_call = tool_call.function %}
                {%- endif %}
                {%- if loop.first %}
                    {%- if content|trim %}
                        {{- '\n\n<tool_call>\n<function=' + tool_call.name + '>\n' }}
                    {%- else %}
                        {{- '<tool_call>\n<function=' + tool_call.name + '>\n' }}
                    {%- endif %}
                {%- else %}
                    {{- '\n<tool_call>\n<function=' + tool_call.name + '>\n' }}
                {%- endif %}
                {%- if tool_call.arguments is defined %}
                    {%- for args_name, args_value in tool_call.arguments|items %}
                        {{- '<parameter=' + args_name + '>\n' }}
                        {%- set args_value = args_value | string if args_value is string else args_value | tojson | safe %}
                        {{- args_value }}
                        {{- '\n</parameter>\n' }}
                    {%- endfor %}
                {%- endif %}
                {{- '</function>\n</tool_call>' }}
            {%- endfor %}
        {%- endif %}
        {{- '<|im_end|>\n' }}
    {%- elif message.role == "tool" %}
        {%- if loop.previtem and loop.previtem.role != "tool" %}
            {{- '<|im_start|>user' }}
        {%- endif %}
        {{- '\n<tool_response>\n' }}
        {{- content }}
        {{- '\n</tool_response>' }}
        {%- if not loop.last and loop.nextitem.role != "tool" %}
            {{- '<|im_end|>\n' }}
        {%- elif loop.last %}
            {{- '<|im_end|>\n' }}
        {%- endif %}
    {%- else %}
        {{- raise_exception('Unexpected message role.') }}
    {%- endif %}
{%- endfor %}
{%- if add_generation_prompt %}
    {{- '<|im_start|>assistant\n' }}
    {%- if enable_thinking is defined and enable_thinking is false %}
        {{- '<think>\n\n</think>\n\n' }}
    {%- else %}
        {{- '<think>\n' }}
    {%- endif %}
{%- endif %}
```


