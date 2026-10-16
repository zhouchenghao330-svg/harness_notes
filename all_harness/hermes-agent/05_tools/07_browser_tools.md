# Hermes Core Tool 07：Browser Tools

> 状态：待共同阅读。

## 1. 工具范围

本组按一套 Stateful Browser Runtime 共同阅读：

~~~text
browser_navigate
browser_snapshot
browser_click
browser_type
browser_scroll
browser_back
browser_press
browser_get_images
browser_vision
browser_console
browser_cdp
browser_dialog
browser_exec
~~~

不为 13 个入口分别建文档，因为它们共享 Browser Session、页面状态和 backend 生命周期。

## 2. 研究定位

~~~text
navigate
   ↓
建立或复用 Browser Session
   ↓
snapshot / vision
   ↓
模型获得页面 Observation 与元素引用
   ↓
click / type / scroll / press / dialog
   ↓
页面状态变化
   ↓
重新观察
~~~

## 3. 共同阅读问题

- Browser Session 由谁创建、复用和关闭？
- backend 如何在内置 Browser、Browser Use、CDP 等实现间选择？
- Snapshot 的元素引用如何生成，页面变化后何时失效？
- 结构化 Snapshot 与 Screenshot / Vision 如何配合？
- click、type、dialog 等副作用是否需要 Approval？
- 登录态、Cookie、下载、上传和跨域数据如何隔离？
- 页面内容怎样防 Prompt Injection？
- Browser 输出如何裁剪、缓存和进入 Context？
- 取消或超时后浏览器动作是否仍可能继续发生？
- `browser_exec` 为什么能替代多个细粒度工具，它的安全边界是什么？

## 4. 主要代码入口

~~~text
tools/browser_tool.py
tools/browser_use_cli.py
tools/browser_cdp_tool.py
tools/browser_dialog_tool.py
tools/browser_supervisor.py
tools/browser_extension_router.py
tools/url_safety.py
~~~
