---
title: Web Search Tools API Key Failover
type: improvement
status: stable
last_updated: 2026-11-01
related_base: agent/offical-tool-list/tools.md
---

## 概述

AstrBot 内置 Web Search Tools 的 API Key 选择与失败重试逻辑已改为 **request-scoped failover**。当某个搜索 Provider 配置了多个 API Key 时，单个 Tool 调用会在遇到 key 级错误后继续尝试下一个未尝试过的 Key，而不是立即失败。

该变更集中在：

- `astrbot/core/tools/web_search_tools.py`
- `_KeyRotator`
- `_tavily_search`、`_tavily_extract`、`_brave_search`、`_firecrawl_search`、`_firecrawl_scrape`、`_exa_search`、`_exa_get_contents`
- `DeepSeekWebSearchTool.call`、`DeepSeekFetchWebPageTool.call`

对外可观察行为是：内置搜索工具在配置多个 Key 时更容忍 Key 级失败；非 Key 级错误仍然 fail-fast，不会进行无意义重试。

## 核心抽象：`_KeyRotator`

`_KeyRotator.get` 的方法签名变为：

```python
async def get(
    self, provider_settings: dict, tried_keys: set[str] | None = None
) -> str:
```

语义从“取当前 cursor 指向的 Key 并前移”变为“**request-scoped 的未尝试 Key 选择器**”。

稳定行为：

1. 在 `self.lock` 保护下，最多扫描 `len(keys)` 次。
2. 使用共享 cursor `self.index` 做 round-robin。
3. 每次取当前 Key 并前移 cursor。
4. 若 `tried_keys is None` 或 Key 不在 `tried_keys` 中，则命中：该 Key 加入 `tried_keys` 并返回。
5. 全部 Key 都已在 `tried_keys` 中时，抛出：

```python
ValueError(f"Error: No untried {self.provider_name} API keys remain.")
```

该设计解决并发/交错请求场景下共享 `self.index` 被推进后，旧实现重试可能取回同一 Key 的问题。`tried_keys` 提供请求级去重，使单次 Tool 调用内不会重复使用同一个 Key。

## Retryable HTTP Status 常量

各 Provider 从共享的 retryable 集合改为独立集合：

```python
_TAVILY_RETRYABLE_HTTP_STATUSES   = frozenset({401, 403, 429, 432, 433})
_FIRECRAWL_RETRYABLE_HTTP_STATUSES = frozenset({401, 402, 403, 429})
_EXA_RETRYABLE_HTTP_STATUSES       = frozenset({401, 402, 429})
_DEEPSEEK_RETRYABLE_HTTP_STATUSES  = frozenset({401, 402, 429})
```

同时保留通用集合：

```python
_RETRYABLE_HTTP_STATUSES = frozenset({401, 403, 429, 432})
_ANYSEARCH_RETRYABLE_HTTP_STATUSES = frozenset({401, 402, 403, 429})
```

这表示不同搜索 Provider 只对自身有意义的 Key 级 HTTP 状态码进行 failover。例如 Tavily 新增 `433`，Firecrawl 新增 `402`，Exa 与 DeepSeek 使用 `401/402/429`。

## 统一 Failover 重试模式

以下函数采用统一结构：

- `_tavily_search`
- `_tavily_extract`
- `_brave_search`
- `_firecrawl_search`
- `_firecrawl_scrape`
- `_exa_search`
- `_exa_get_contents`
- `DeepSeekWebSearchTool.call`
- `DeepSeekFetchWebPageTool.call`

统一结构可概括为：

```python
last_error = None
tried_keys: set[str] = set()
for _ in range(len(set(keys))):
    key = await _XXX_KEY_ROTATOR.get(provider_settings, tried_keys)
    # 发请求
    if response.status != 200:
        last_error = Exception(...)
        if response.status in _XXX_RETRYABLE_HTTP_STATUSES:
            continue
        raise last_error
    # 解析 success payload 并 return

if last_error is not None:
    raise last_error
raise Exception("... failed with all configured keys.")
```

关键行为：

- 重试上限由 `len(keys)` 改为 `len(set(keys))`，重复配置的 Key 不消耗重试配额。
- `tried_keys` 是单次调用内的局部集合，不会跨调用泄漏。
- 成功响应会解析 payload 并直接返回归一化结果。
- Key 级错误记录为 `last_error` 后继续尝试下一个未尝试 Key。
- 非 Key 级错误立即抛出，不进行重试。
- 所有 Key 耗尽后抛出最后一个错误或聚合错误。

DeepSeek 的两个 Tool 使用 `for…else` 变体：成功解析 `data` 后 `break`，`else` 分支统一抛聚合错误。

## Brave 422 特判

`_brave_search` 中 `422` 不在通用 retryable 集合里，需要解析响应体判定：

```python
retryable = response.status in _RETRYABLE_HTTP_STATUSES
if response.status == 422:
    try:
        error = json.loads(reason).get("error", {})
    except (json.JSONDecodeError, AttributeError):
        error = {}
    retryable = isinstance(error, dict) and \
                error.get("code") == "SUBSCRIPTION_TOKEN_INVALID"
```

稳定行为：

- 仅 `SUBSCRIPTION_TOKEN_INVALID` 视为 Key 级错误，可换 Key 重试。
- 其余 `422`，例如 `VALIDATION`、`error:null`、非 JSON 响应，直接抛出，不换 Key。

## Exa Key 缺失校验

Exa 的两个函数从单次请求重构为完整 failover 循环，并新增 `websearch_exa_key` 缺失检查。缺失时抛出：

```python
ValueError("Error: Exa API key is not configured in AstrBot.")
```

这使 Exa 搜索在未配置 Key 时给出明确配置错误，而不是进入无意义请求或隐式失败。

## 配置项与作用范围

摘要列出的配置项均支持多 Key failover：

- `websearch_tavily_key`
- `websearch_bocha_key`
- `websearch_brave_key`
- `websearch_firecrawl_key`
- `websearch_exa_key`
- `websearch_deepseek_key`

该行为作用于 AstrBot 内置 Web Search Tools 的调用链路，不是新的插件装饰器或消息组件契约。

## 数据流

```text
Tool.call / _xxx_search(provider_settings, payload)
  -> 检查 keys 非空（否则 ValueError / 配置错误字符串）
  -> tried_keys = set()
  -> for i in range(len(set(keys))):
        key = _XXX_KEY_ROTATOR.get(provider_settings, tried_keys)
              -> lock -> 扫描共享 cursor -> 跳过 tried_keys -> 命中并登记
        HTTP 请求
          -> 成功 -> 返回归一化结果
          -> status 属于 provider retryable -> last_error = ..., continue
          -> 其他 -> raise last_error（fail-fast）
  -> 耗尽 -> raise last_error 或聚合错误
```

`_KeyRotator` 从纯 round-robin 变成**并发安全的 request-local 去重游标**。共享 cursor 负责在多个请求之间轮转 Key；`tried_keys` 负责保证单个请求内不重复尝试同一个 Key。

## 变更影响分析

- **多 Key 配置语义增强**：`websearch_tavily_key`、`websearch_bocha_key`、`websearch_brave_key`、`websearch_firecrawl_key`、`websearch_exa_key`、`websearch_deepseek_key` 不再只是简单轮转来源，而是在单次 Tool 调用内具备 Key 级 failover 能力。
- **Key 级错误与业务错误边界更清晰**：`401/402/403/429/432/433` 以及 Brave 的 `SUBSCRIPTION_TOKEN_INVALID` 等 Key 级错误会自动尝试下一个未尝试 Key；`5xx`、参数校验错误、其他 `422` 等非 Key 级错误立即抛出。AI 助手在分析搜索失败时，应先区分 HTTP 状态是否属于对应 Provider 的 retryable 集合。
- **并发请求不再重复消耗同一 Key**：共享 `self.index` 会因其他并发请求推进，但 `tried_keys` 保证当前请求只尝试未尝试过的 Key。插件或 Tool 调用方不应假设重试一定按配置数组顺序从第一个 Key 开始。
- **重复配置 Key 不增加重试次数**：重试上限为 `len(set(keys))`。如果同一 Key 被重复写入配置，不会因此获得额外重试配额。
- **Exa 未配置 Key 时显式报错**：`_exa_search` 与 `_exa_get_contents` 会抛出 `ValueError("Error: Exa API key is not configured in AstrBot.")`。AI 开发者在排查 Exa 搜索失败时，应优先确认 `websearch_exa_key` 是否配置。
- **Brave 422 不是通用可重试错误**：只有响应体解析出的 `error.code == "SUBSCRIPTION_TOKEN_INVALID"` 才触发换 Key。其他 `422` 不应被理解为 Key 失效。
- **DeepSeek search 与 fetch 行为一致**：`DeepSeekWebSearchTool.call` 与 `DeepSeekFetchWebPageTool.call` 都覆盖 Key failover 与耗尽路径，二者在 Key 级错误处理上保持同一模式。
- **不改变统一消息模型**：本变更没有新增 Message Chain 组件、Provider 调用接口或 Platform Adapter 接口。其影响集中在内置 Web Search Tools 的配置消费、HTTP 请求重试和错误语义。
- **耗尽错误保留最后失败原因**：所有 Key 尝试完毕后，调用方会收到最后一个 `last_error` 或聚合错误。AI 助手生成诊断逻辑时，应预期异常消息可能来自最后一次尝试，而不是第一个 Key。