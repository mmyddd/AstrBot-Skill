---
title: OpenAI Responses API 无状态 Provider 适配器
type: feature
status: stable
last_updated: 2026-08-02
related_base: agent/Invoke-llm.md
---

## 概述

AstrBot 新增 `ProviderOpenAIResponses` 适配器，用于将 AstrBot 的 Chat Provider 调用链接入 OpenAI 的 Responses API，并同时兼容 xAI 与 DeepSeek 的 Responses 实现。该适配器通过 `@register_provider_adapter("openai_responses", ...)` 注册，继承自 `ProviderOpenAIOfficial`，本质是一个无状态（stateless）Responses API 适配器。

此次变更对第三方开发者最重要的影响是：Provider 注册表新增了一个可用的 provider type `openai_responses`，默认配置中新增了预置的 OpenAI Responses 与 DeepSeek Responses 提供商，并且 xAI 的默认 provider type 从 `xai_chat_completion` 切换为 `openai_responses`。这不改变插件调用 LLM 的入口（`context.llm_generate(...)`、`context.tool_loop_agent(...)` 等），但会影响 Provider ID 选择、模型输出中的推理内容形态，以及多轮工具调用历史的回放方式。

## 核心行为

### 1. Provider 注册与预置配置

`ProviderManager` 新增 `openai_responses` case，动态导入：

```python
astrbot.core.provider.sources.openai_responses_source.ProviderOpenAIResponses
```

默认配置新增两个预置提供商：

- OpenAI Responses：`id="openai-responses"`，`provider="openai"`，`type="openai_responses"`，`api_base="https://api.openai.com/v1"`
- DeepSeek Responses：`id="deepseek-responses"`，`provider="deepseek"`，`type="openai_responses"`，`api_base="https://api.deepseek.com/v1"`

xAI 的 provider `type` 由 `xai_chat_completion` 改为 `openai_responses`，并移除了 `xai_native_search` 配置字段。`xai_native_search` 显示条件被收紧为 `{"provider": "xai", "type": "xai_chat_completion"}`，因此该开关只在旧版 xAI Chat Completions 模式下生效，xAI Responses 模式不再原生搜索配置。

### 2. 核心类型变更

`LLMResponse.raw_completion` 的类型联合扩展为：

```python
ChatCompletion | Response | GenerateContentResponse | AnthropicMessage | None
```

这意味着插件或下游逻辑在检查 `raw_completion` 时，可能需要处理 OpenAI Responses 的 `openai.types.responses.Response` 对象。

### 3. 上下文转换：Chat Completions 格式 → Responses `input` items

核心转换方法是：

```python
_convert_chat_messages_to_response_input(messages: list[dict]) -> list[dict]
```

它将 AstrBot 上下文（OpenAI Chat Completions 格式）转换为 Responses API 的 `input` items。转换规则包括：

- `role == "tool"` 的消息转换为 `{"type": "function_call_output", "call_id", "output"}`；当 content 不是字符串时，会进行 JSON 序列化。
- `user`、`assistant`、`system`、`developer` 角色转换为 `{"type": "message", "role", "content"}`。
- `think` 段特殊处理：如果 `part["encrypted"]` 是 JSON 且类型为 `_REASONING_STATE_TYPE`，则反序列化恢复为原始 reasoning items；否则，如果 Provider 是 DeepSeek，则重建为 `{"type": "reasoning", "content": [{"type": "reasoning_text", "text": think}]}`。
- `text` 段：assistant 角色直接拼接为字符串；其他角色转为 `input_text`。
- 非 assistant 角色的 `image_url` 段转为 `input_image`，并保留 `detail` 合法性校验。
- `audio_url` / `input_audio` 段：assistant 角色以 `[Audio]` 占位，其他角色转为 `input_text`。
- assistant 消息的 `tool_calls` 转换为 `{"type": "function_call", "call_id", "name", "arguments"}`，并保留原始 id。
- reasoning items 会按原始顺序插入到消息之前。

### 4. Payload 构建与无状态强制

`_prepare_chat_payload(...)` 构建请求 payload，核心结构为：

```python
{"input": converted_input, "model": ..., "store": False}
```

有 `system_prompt` 时写入 `instructions`。如果存在图片，会调用 `_materialize_context_image_parts` 进行物化。

`_query(...)` 与 `_query_stream(...)` 会强制移除 `previous_response_id` 和 `conversation`，并强制 `store=False`。这是该适配器被称为“无状态 Responses API 适配器”的根本原因：每次请求都携带完整历史，不依赖服务端会话状态。

### 5. 参数规范化

`_query(...)` 在处理请求参数时，会检查 `inspect.signature(self.client.responses.create).parameters` 中的 `default_params`：

- 不在 `default_params` 中的字段统一放入 `extra_body`。
- `max_tokens` 映射为 `max_output_tokens`（当未显式设置时）。
- `reasoning_effort` 映射为 `reasoning: {"effort": ...}`。
- 校验返回类型必须是 `openai.types.responses.Response`。

### 6. 流式事件映射

`_query_stream(...)` 将 Responses API 的流式事件映射为 AstrBot 的 `LLMResponse` chunk：

- `response.output_text.delta` 与 `response.refusal.delta` 产生普通文本 chunk（`Comp.Plain`，`is_chunk=True`）。
- `response.reasoning_text.delta` 与 `response.reasoning_summary_text.delta` 产生 `reasoning_content` chunk。
- `response.completed`、`response.incomplete`、`response.failed` 产生最终完整 `LLMResponse` 后结束。
- `error` 事件抛出 `RuntimeError`，携带 `code`、`message`、`response_id`。
- 无终止事件时抛出 `EmptyModelOutputError`。

### 7. 响应归一化

`_parse_response(response: Response, tools: ToolSet | None) -> LLMResponse` 负责把 Responses API 响应归一化为 `LLMResponse`：

- `status == "failed"` 或 `incomplete_details.reason == "content_filter"` 时抛出 `RuntimeError`。
- 文本内容从 `output[*].message.content` 的 `output_text` 或 `refusal` 中提取。
- reasoning items 被序列化为 `reasoning_signature`，JSON 包裹形式为 `{"type": "openai_responses_reasoning", "items": [...]}`，并拼接 `reasoning_content`（优先 `reasoning_text`，缺失时使用 summary）。
- `function_call` items 填充 `tools_call_args`、`name`、`ids`；存在时设置 `role="tool"`。
- `usage` 透传。

## 数据流 / 调用链

```
ProviderManager.dynamic_import_provider("openai_responses")
  → ProviderOpenAIResponses(ProviderOpenAIOfficial)
    → _prepare_chat_payload(...)
      → _convert_chat_messages_to_response_input(context_query)
    → _query/_query_stream → client.responses.create(..., stream=...)
      → _parse_response(response, tools) → LLMResponse
```

回放链路的关键设计是：AstrBot 上下文中的 `think` 段 `encrypted` 字段保存了 `_REASONING_STATE_TYPE` JSON，新请求时反序列化还原为 Responses 的 `reasoning` items，从而保证多轮 function call 历史完整回放。由于该适配器无服务端状态，所有上下文必须由客户端通过 `input` 完整携带。

## 变更影响分析

- **Provider ID 选择变化**：默认配置中出现了 `openai-responses` 与 `deepseek-responses` 两个新预置 Provider ID。AI 开发者在生成插件配置或提示用户选择 Provider 时，不应再把 xAI 默认当作 Chat Completions 类型；xAI 的默认 type 已是 `openai_responses`。
- **`xai_native_search` 仅在旧模式生效**：如果用户仍使用旧版 xAI Chat Completions，`xai_native_search` 才可见。基于该配置项开发的功能应检查用户实际使用的 provider type，避免假设 xAI 一定携带原生搜索开关。
- **多轮工具调用不依赖服务端状态**：Responses API 适配器强制 `store=False` 并移除 `previous_response_id`、`conversation`。插件作者不应假设 Responses Provider 会记住远程会话；如需跨轮上下文，必须依赖 AstrBot 的 ConversationManager 与请求时的完整历史传递。
- **推理内容形态更明确**：`LLMResponse` 现在可能包含 `reasoning_content` 与 `reasoning_signature`。`reasoning_signature` 是 JSON 包裹的 `openai_responses_reasoning` items，AI 开发者在分析模型输出时应区分普通文本、reasoning 文本与推理签名数据，不要把 reasoning 内容当作最终业务回复。
- **工具调用历史回放依赖 `think` 段加密字段**：AstrBot 上下文中的 `think` 段 `encrypted` 字段保存了 `_REASONING_STATE_TYPE` JSON。自定义 Tool 或插件如果手动构造上下文，不应随意删除或改写 `think` 段的 `encrypted` 字段，否则可能破坏 Responses Provider 的推理历史回放。
- **参数映射存在 Chat API 兼容层**：`max_tokens` 会被映射为 `max_output_tokens`，`reasoning_effort` 会映射为 `reasoning.effort`。插件通过 `tool_loop_agent(...)` 或其他 Provider 调用传入参数时，应使用 AstrBot 侧已有的参数名，而不应直接传递 Responses API 专属参数名；适配器会处理映射。
- **流式输出事件与终止语义**：流式模式下，`response.completed`、`response.incomplete`、`response.failed` 都会产生最终 `LLMResponse`。插件应把 `incomplete` 视为正常终止路径之一；`error` 事件会以携带 `response_id` 的 `RuntimeError` 形式暴露。AI 助手在排查流式中断时，应结合 `response_id` 定位问题。
- **`raw_completion` 类型扩展**：读取 `LLMResponse.raw_completion` 的现有插件可能需要增加对 `Response` 对象的处理分支。不应假设 `raw_completion` 一定是 Chat Completions 的 `ChatCompletion`。
- **无新增插件侧入口**：该变更没有新增装饰器、Message Chain 组件、`AstrMessageEvent` 方法或 Platform Adapter 接口。插件仍通过 `context.llm_generate(...)`、`context.tool_loop_agent(...)`、`context.get_current_chat_provider_id(...)` 等现有契约使用 LLM 能力。
- **DeepSeek 推理内容兼容**：当没有 `encrypted` 状态且 Provider 为 DeepSeek 时，`think` 文本会重建为 `reasoning` item。这意味着 DeepSeek Responses 的推理内容可以跨轮传递，但该行为依赖适配器对 provider 类型的判断。AI 开发者在设计 Agent 历史分析逻辑时，应预期 DeepSeek 的 reasoning 文本可能以 `reasoning` item 形态进入下一轮请求。