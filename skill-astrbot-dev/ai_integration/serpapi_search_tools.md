---
title: SerpApi 搜索工具接入
 type: feature
status: stable
last_updated: 2026-10-17
related_base: "agent/register tools.md"
---

## 概述

AstrBot 新增 SerpApi 搜索提供商，并在核心内置工具层注册 4 个 `FunctionTool[AstrAgentContext]` 子类，用于向 Agent 提供 Web、图片、财经和学术搜索能力。这些工具通过 `@builtin_tool(config=_SERPAPI_WEB_SEARCH_TOOL_CONFIG)` 与 `@pydantic_dataclass` 挂载，属于 Agent Tool 能力，不新增 Message Chain 组件，也不改变 Provider、Adapter 或插件装饰器接口。

工具定义位于 `astrbot/core/tools/web_search_tools.py`。当 `provider_settings.websearch_provider` 为 `serpapi` 时，主 Agent 的 `_apply_web_search_tools` 会一次性注册全部 4 个 SerpApi 工具。

## 工具契约

| 类名 | 工具名 | SerpApi engine | 主要参数 |
|---|---|---|---|
| `SerpApiWebSearchTool` | `web_search_serpapi` | `google` | `query` 必填；`max_results` 取值 1-10，默认 5；`country` 映射为 `gl`；`search_lang` 映射为 `hl`；支持 `location` |
| `SerpApiImageSearchTool` | `image_search_serpapi` | `google_images_light` | `query`；`max_results` 取值 1-10；`country` 映射为 `gl`；`search_lang` 映射为 `hl` |
| `SerpApiFinanceSearchTool` | `finance_search_serpapi` | `google_finance` | `query` 为 ticker，例如 `AAPL:NASDAQ`；`search_lang` 映射为 `hl`；`window` 取值为 `1D`、`5D`、`1M`、`6M`、`YTD`、`1Y`、`5Y`、`MAX` |
| `SerpApiScholarSearchTool` | `scholar_search_serpapi` | `google_scholar` | `query`；`max_results` 取值 1-20；`search_lang`；`year_from` 映射为 `as_ylo`；`year_to` 映射为 `as_yhi` |

上述工具名同时加入 `WEB_SEARCH_TOOL_NAMES` 与 `__all__`，并参与引用提示词触发条件。

## 配置契约

SerpApi 提供商由以下配置控制：

- `provider_settings.web_search == true`
- `provider_settings.websearch_provider == "serpapi"`
- `provider_settings.websearch_serpapi_key` 为非空 Key 列表

`_SERPAPI_WEB_SEARCH_TOOL_CONFIG` 将工具启用条件固定为 `{"provider_settings.web_search": true, "provider_settings.websearch_provider": "serpapi"}`。默认配置新增 `"websearch_serpapi_key": []`，字段类型为 `list[string]`，并标记为 `secret: True`。`normalize_legacy_web_search_config` 的“字符串 → 列表”归一化字段集中也加入 `websearch_serpapi_key`，与既有搜索 Key 行为保持一致。

## 请求与 Key 轮换

核心请求入口为：

```python
async def _serpapi_request(provider_settings, payload, *, engine) -> dict
```

请求目标为 `https://serpapi.com/search.json`，参数由 `{**payload, engine, api_key}` 组成，超时时间为 60 秒。Key 管理使用：

```python
_SERPAPI_KEY_ROTATOR = _KeyRotator("websearch_serpapi_key", "SerpApi")
```

请求按 Key 数量循环，每轮通过 `_SERPAPI_KEY_ROTATOR.get()` 获取 Key。HTTP 200 时解析 JSON：若 `search_metadata.status == "Error"`，或存在 `error` 且 `status != "Success"`，会抛出 `RuntimeError`；空结果保持正常返回。错误字符串中的 API Key 会替换为 `[redacted]`。

非 200 响应会记录 `last_error`：`401`、`403`、`429` 会继续换 Key；其他状态码立即抛出。`aiohttp.ClientError` 与 `asyncio.TimeoutError` 会转换为无 cause 的 `RuntimeError`（`from None`），因为异常字符串可能包含带 Key 的 URL。所有 Key 耗尽后抛出 `last_error`，兜底抛出 `"SerpApi search failed with all configured keys."`。

当 `provider_settings["websearch_serpapi_key"]` 为空时，工具返回 Error 字符串，不抛出异常。

## 搜索结果映射

```python
async def _serpapi_search(provider_settings, payload) -> list[SearchResult]
```

`_serpapi_search` 固定使用 `engine="google"`，将 `organic_results` 映射为 `SearchResult(title, url=link, snippet, favicon)`，并过滤没有 `link` 的结果项。

图片搜索结果项包含 `title`、`url`（来源页）、`snippet`（来源名）、`image_url`、`thumbnail_url`、`width`、`height`、`index`。当 `original` 不是 `http(s)` 时降级使用 `thumbnail`；两者都不是 `http(s)` 时丢弃该项。

财经搜索输出 `{results, summary, knowledge_graph, financials, news_results, suggestions}`。存在 `summary` 时注入一条引用，`url` 取 `search_metadata.google_finance_url`，否则回退为 `https://www.google.com/finance/quote/{quote(query, safe=':-.')}`。仅当指定 `window` 时追加 `graph` 与 `key_events`。

学术搜索的 `url` 依次取 `link`、首个 `resources[].link`，兜底为 `https://scholar.google.com/scholar?q={quote(title or query)}`。输出包含 `publication_info`、`cited_by_count`、`cited_by_url`（源自 `inline_links.cited_by`）与 `resources`。

图片与学术搜索使用 `ref_uuid = str(uuid.uuid4())[:4]` 作为批次前缀，引用索引形如 `"a1b2.1"`；财经搜索固定使用 `"{uuid4[:4]}.1"` 形式。

## 参数校验

以下情况返回错误字符串而非异常：`query` 为空、`window` 非法、年份超出 1-9999、`year_from > year_to`。

## 与统一消息模型的关系

SerpApi 工具只扩展 Agent 可调用的 Tool 集合，不新增 Message Chain 组件。工具调用结果以 `SearchResult`、结构化字典或错误字符串进入 Agent 上下文。插件作者仍应通过 `AstrMessageEvent`、`MessageChain`、`context.tool_loop_agent(...)` 等既有抽象使用 Agent 能力，不应直接依赖 SerpApi 原始 HTTP 响应结构。

## 变更影响分析

- **新增稳定的内置工具名**：`web_search_serpapi`、`image_search_serpapi`、`finance_search_serpapi`、`scholar_search_serpapi` 会作为 LLM tool_call 名称出现。AI 开发者在分析工具调用日志时，应把这些名称识别为 SerpApi 搜索工具，而不是插件自定义工具。
- **启用条件严格绑定 provider 配置**：只有 `provider_settings.web_search == true` 且 `provider_settings.websearch_provider == "serpapi"` 时，主 Agent 才会注册全部 4 个 SerpApi 工具。仅配置 Key 而未切换 `websearch_provider` 不会触发注入。
- **缺少 Key 是工具级错误而非系统异常**：`websearch_serpapi_key` 为空时工具返回 Error 字符串，因此 Agent 可能继续推理或向用户暴露该错误文本。插件或上层逻辑不应假设缺少 Key 会导致 `RuntimeError`。
- **Key 轮换只对认证/限流状态重试**：`401`、`403`、`429` 会切换到下一个 Key；其他非 200 状态码立即失败。AI 开发者在排查 SerpApi 失败时，应区分 Key 轮换耗尽与不可重试的请求错误。
- **空搜索结果是正常返回**：HTTP 200 且无错误状态时，空结果不会抛异常。Agent 应能处理“搜索成功但结果为空”的返回值。
- **引用索引前缀不稳定**：图片、学术与财经搜索生成的 `ref_uuid` 每次调用可能不同，`"a1b2.1"` 一类索引只用于当前批次引用，不应被插件当作持久业务 ID 或跨调用关联键。
- **图片结果存在显式降级与丢弃规则**：当原图 URL 非 HTTP(S) 时会回退到缩略图；两者都不可用时该项被丢弃。AI 开发者在消费图片搜索输出时，应预期结果数量可能少于请求的 `max_results`。
- **财经与学术搜索有专用输出契约**：财经搜索可能包含 `summary`、`financials`、`news_results`、`suggestions`、`graph`、`key_events`；学术搜索包含 `publication_info`、引用统计与 `resources`。这些结构不同于普通 Web 搜索的 `SearchResult` 列表，插件不应统一按普通搜索结果解析。
- **不新增插件装饰器或 Adapter 接口**：本变更是核心内置 Tool 与 Provider 配置扩展，没有新增 Message Chain 组件、Platform Adapter 方法或插件生命周期钩子。插件仍应通过既有 Agent Tool 抽象接入或调用搜索能力。
- **不应绕过工具抽象直接调用 SerpApi**：工具已封装 Key 轮换、错误脱敏、超时和结果映射。插件若直接调用 SerpApi HTTP 接口，将失去统一的 Key 管理与引用格式约定。