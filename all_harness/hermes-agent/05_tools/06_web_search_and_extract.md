# Hermes Core Tool 06：web_search / web_extract

> 状态：待共同阅读。

## 1. 研究定位

这两个工具构成 Hermes 的只读 Web 信息获取链：

~~~text
web_search(query)
  → 找到候选来源

web_extract(url)
  → 获取指定页面内容

外部网页文本
  → 安全包装、截断与结果治理
  → 下一次模型调用
~~~

重点不是搜索 Provider 的产品功能，而是外部不可信内容怎样成为 Agent Observation。

## 2. 共同阅读问题

- 模型侧 Schema 如何区分搜索和定向提取？
- Provider 如何选择，失败时是否 fallback？
- URL 有哪些协议、内网和 SSRF 安全检查？
- 网页正文如何清洗，引用和来源信息怎样保留？
- Prompt Injection 内容如何标记为不可信数据？
- 大结果如何截断、缓存、外溢和继续读取？
- 搜索和提取的 timeout、取消和重试语义是什么？
- Tool Result 哪些内容进入模型，哪些只供 UI / trace 使用？
- 它们与 Browser Tools 的职责边界是什么？

## 3. 主要代码入口

~~~text
tools/web_tools.py
tools/web_result_cache.py
tools/url_safety.py
tools/website_policy.py
tools/read_extract.py
~~~
