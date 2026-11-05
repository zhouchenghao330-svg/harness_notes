# 02 Context And Prompt Assembly

## 这个组件研究什么

模型每一轮到底看到了什么上下文，以及这些上下文是怎么拼出来的。

## 应该回答的问题

- system prompt 从哪些部分组成
- session history 如何裁剪
- memory / recall / plugin context 怎么注入
- 为什么某些内容放 system prompt，某些内容放 user message
- compaction / compression 怎么影响上下文

## 典型关注点

- prompt builder
- system prompt assembly
- context compressor
- ephemeral context injection
