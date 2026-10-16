# Hindsight 入口 Bank 和请求上下文

> 组件 01：已共同阅读。记录日期：2026-09-30。
>
> 本文记录 HTTP 与 MCP 请求进入 MemoryEngine 时，如何确定 bank、传递身份和取消信号。以下行为已由当前源码确认，细节位置见文末。

## 三个概念

`tenant/schema` 决定本次调用使用哪片数据库空间；`bank_id` 决定访问其中哪个记忆库；`RequestContext` 携带调用者身份、租户认证结果与可选的取消信号。

`bank_id` 是引擎方法的参数。`RequestContext` 是请求的执行上下文，与 Reflect 发给模型的消息和 Prompt 分属不同用途。

## 入口运行链

~~~text
HTTP URL bank_id -> resolve alias --+
                                    +-> MemoryEngine -> operation
MCP path/header/tool -> resolve bank-+       |
                                            v
                                    authenticate tenant/schema
                                    validate operation
                                    read/write bank data
~~~

HTTP 的核心接口是：

- Retain：`POST /v1/default/banks/{bank_id}/memories`
- Recall：`POST /v1/default/banks/{bank_id}/memories/recall`
- Reflect：`POST /v1/default/banks/{bank_id}/reflect`

HTTP 路由先解析 bank 别名，使后续处理得到正式 ID。认证信息从 Authorization 请求头进入 `RequestContext`；该对象保存在本次请求的 state 中复用，认证后的 schema 也会缓存到该对象上。

MCP 的连接路径可以绑定单个 bank；多 bank 模式还可从请求头和工具参数确定 bank。MCP 工具根据传输层保存的信息构造 `RequestContext`，随后直接调用同一个 MemoryEngine。

## 身份与权限

租户扩展负责认证并返回数据库 schema。默认的 `DefaultTenantExtension` 使用配置的 schema，接受所有请求；内置 `ApiKeyTenantExtension` 检查配置的 API key。自定义租户扩展可以根据调用者选择不同 schema。

操作权限可由可选的 `OperationValidatorExtension` 检查。HTTP 核心入口还挂有预检查和请求准入机制，引擎内则调用对应的 `validate_retain()`、`validate_recall()` 或 `validate_reflect()`。

因此，传入一个 `bank_id` 只是在选记忆库。调用者是否有访问权限取决于部署所配置的认证和操作校验扩展。

## bank 不存在时的行为

| 操作 | 当前代码行为 |
| --- | --- |
| Retain | 可按需创建 bank，然后写入 |
| Recall | 检查 bank 是否存在，不存在返回 404 |
| Reflect | 调用 `ensure_bank_profile()`，缺少 bank 时也可能创建默认 profile |

Reflect 使用只读检索工具，但其入口对 bank profile 的处理可能产生写入，不能据此把整个调用说成绝无写入。

## 取消信号

HTTP Recall 与 Reflect 的断开检测由 `ClientDisconnectCancellationMiddleware` 实现。客户端断开时，它触发请求 token；handler 将 token 放入 `RequestContext`，引擎在流水线阶段边界或循环迭代边界调用 `raise_if_cancelled()`。取消异常在 HTTP 层映射为 499。

这条断开检测仅覆盖上述 HTTP 读接口。MCP 和 Retain 没有通过同一个中间件路径获得该断开取消机制。

## 源码证据

路径相对 `hindsight-api-slim/hindsight_api/`：

- `models.py`：`RequestContext` 与 `raise_if_cancelled()`。
- `api/http.py`：`get_request_context()`、`precheck_for()`、`admit_for()`、三个 handler、`run_cancellable_on_disconnect()`。
- `api/unknown_params.py`：`UnknownParamsRoute._resolve_bank_alias()`。
- `api/mcp.py` 与 `mcp_tools.py`：MCP bank 解析与 `_get_request_context()`。
- `engine/memory_engine.py`：`_authenticate_tenant()`、`resolve_bank_alias()`、`_ensure_bank_exists()`、`_require_bank_exists()`、`ensure_bank_profile()`。
- `extensions/builtin/tenant.py`：默认租户和 API key 认证实现。
- `api/disconnect.py`：客户端断开与取消 token。

下一组件：02 Retain 写入流水线，见 [[01_components|组件索引]]。
