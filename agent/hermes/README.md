# Hermes Memory 拆解

这个目录用于拆解 Hermes Agent 的长期记忆体系，重点不是复述 README，而是回答三个问题：

1. 它到底有哪些记忆层。
2. 每一层的写入、读取、注入时机是什么。
3. 哪些能力是本地单机完成，哪些能力依赖外部 provider 或云端服务。

当前初步结论：

- 默认长期记忆是本地主导，不是纯云上黑盒。
- Hermes 把长期记忆拆成小而常驻的显式记忆、可检索的全量会话历史、以及后台 review 驱动的主动整理。
- 外部 memory provider 是增强层，不是默认基础层。

建议阅读顺序：

1. [00-overview.md](00-overview.md)
2. [01-built-in-memory.md](01-built-in-memory.md)
3. [02-session-recall.md](02-session-recall.md)
4. [03-background-review.md](03-background-review.md)
5. [04-memory-providers.md](04-memory-providers.md)
