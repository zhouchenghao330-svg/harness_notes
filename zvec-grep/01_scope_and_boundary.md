# zvec-grep 01：研究边界

## 1. 纳入范围

只要代码会改变 Agent 下一步看到什么、能调用什么、检索什么或是否继续等待，就纳入：

- Agent 安装器和配置写入
- MCP Server、工具定义和工具说明
- 查询输入规范化和检索路由
- 工作区索引、代码/文档结构抽取
- 搜索召回、融合、去重和结果选择
- Agent Context 输出格式
- 索引新鲜度、后台刷新和 watcher
- Remote Embedding 授权和本地 FTS fallback

## 2. 暂时排除

- 与 Agent 行为无关的 CLI 展示
- zvec 底层实现细节
- Benchmark 的完整执行框架
- 普通安装、发布和工程基础设施
- 不影响检索输入输出的通用工具函数

## 3. 与 Agent Harness 的关系

```text
Agent Harness
  └─ 调用外部能力：zvec-grep

zvec-grep
  └─ 提供 Workspace Retrieval / Evidence Context
```

## 4. 需要共同确认的问题

- zvec-grep 的“Agent 行为”主要由 MCP tool description、server instructions 还是安装后的 AGENTS 指导决定？
- 自动刷新算不算搜索工具的外部副作用？
- 索引实体和最终 Agent Context 之间，哪一层才是主要的语义边界？
- `zvec_grep_search` 是否应该被视为 Tool，还是一个独立的 Retrieval Runtime？

