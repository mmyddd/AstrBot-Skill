---
title: 多模态上下文 Token 估算
type: improvement
status: stable
last_updated: 2026-06-24
related_base: agent/context-management.md
---

## 概述

AstrBot 的 `EstimateTokenCounter` 现在会在上下文 token 估算中计入多模态内容。此前估算逻辑只处理字符串内容和 `TextPart`，当对话历史中包含图片、音频或思考内容时，这些内容不会进入 token 预算，可能导致上下文压缩触发过晚，并在 Provider 请求阶段先遇到上下文长度错误。

本次变更位于 Agent 上下文管理链路中的 token 估算阶段，影响的是请求 LLM 前的上下文预算判断与压缩触发时机，不改变插件构造 Message Chain、调用 `context.llm_generate(...)`、调用 `context.tool_loop_agent(...)` 或使用 Provider 的方式。

## 核心行为

实现位置：

```python
astrbot/core/agent/context/token_counter.py
```

`EstimateTokenCounter` 在处理 `Message.content` 为列表的多模态内容时，新增对以下 ContentPart 的估算：

- `TextPart`：继续按文本内容估算 token。
- `ThinkPart`：按 `think` 字段中的文本内容估算 token。
- `ImageURLPart`：每个图片内容按固定估算值计入。
- `AudioURLPart`：每个音频内容按固定估算值计入。

新增估算常量：

```python
IMAGE_TOKEN_ESTIMATE = 765
AUDIO_TOKEN_ESTIMATE = 500
```

稳定行为可以概括为：

```python
if isinstance(part, TextPart):
    total += self._estimate_tokens(part.text)
elif isinstance(part, ThinkPart):
    total += self._estimate_tokens(part.think)
elif isinstance(part, ImageURLPart):
    total += IMAGE_TOKEN_ESTIMATE
elif isinstance(part, AudioURLPart):
    total += AUDIO_TOKEN_ESTIMATE
```

这意味着同一条消息中如果包含多个图片或音频片段，每个片段都会独立计入 token 预算。例如三张图片会产生 `IMAGE_TOKEN_ESTIMATE * 3` 的估算开销。

## 与上下文压缩数据流的关系

该变更发生在 Agent 上下文管理的估算层。典型数据流为：

1. 插件或主 Agent 通过 `context.tool_loop_agent(...)`、Provider 调用或会话历史机制形成 `Message` 列表。
2. `Message.content` 可能是纯字符串，也可能是由 `TextPart`、`ImageURLPart`、`AudioURLPart`、`ThinkPart` 等组成的列表。
3. 上下文管理逻辑调用 `EstimateTokenCounter.count_tokens(...)` 估算当前上下文大小。
4. 图片、音频和思考内容现在会进入估算结果。
5. 当估算结果更早达到上下文限制时，已有的上下文压缩或截断策略可以更及时触发。
6. 压缩后的上下文再继续进入 LLM Provider 请求流程。

该设计降低了多模态消息与上下文压缩策略之间的脱节：Provider 层实际会为图片和音频消耗上下文窗口，而 token 估算层现在也会保守地反映这部分成本。

## trusted_token_usage 优先级

`EstimateTokenCounter.count_tokens(...)` 仍保留可信 token 用量优先的行为。当调用方传入 `trusted_token_usage` 时，方法直接返回该值，不再对文本、图片、音频、思考内容或 tool calls 做估算。

因此，多模态估算只影响没有可信 Provider token usage 可用时的本地估算路径。若 Provider 或上游能力已经返回准确 token 数，AstrBot 会优先使用真实用量。

## 对插件开发者的影响

插件作者通常不需要直接调用 `EstimateTokenCounter`。影响主要体现在使用多模态上下文的 Agent 或 LLM 调用中：

```python
llm_resp = await self.context.tool_loop_agent(
    event=event,
    chat_provider_id=await self.context.get_current_chat_provider_id(event.unified_msg_origin),
    prompt="请分析这张图片和相关语音内容",
    image_urls=["https://example.com/image.png"],
)
```

当上下文历史中包含图片、音频或思考内容时，AstrBot 现在更可能在 Provider 报错前触发上下文压缩。插件侧的调用接口不变，但长对话、多图输入或音频输入场景下，历史保留长度可能比过去更保守。

## 变更影响分析

- **多模态上下文更容易提前压缩**：图片按 `765` token、音频按 `500` token 计入估算后，包含多张图片或音频的会话会更快达到上下文阈值。这是为了避免低估 token 导致 Provider 返回上下文长度错误。
- **估算值是保守近似，不是 Provider 精确计费值**：`IMAGE_TOKEN_ESTIMATE` 和 `AUDIO_TOKEN_ESTIMATE` 是本地估算值。AI 开发者不应把它们理解为所有模型的真实 token 消耗，也不应用它们计算费用。
- **多图消息按数量线性累加**：每个 `ImageURLPart` 都会增加 `765` token。生成插件或 Agent 代码时，如果允许用户一次性提交大量图片，应预期上下文压缩会更频繁触发。
- **音频内容不再是零成本上下文**：`AudioURLPart` 现在会增加 `500` token。处理语音或音频 URL 的插件不应假设音频只影响附件处理而不影响上下文预算。
- **ThinkPart 会按文本估算**：思考内容现在按 `think` 文本进入 token 估算。如果系统保留了较长的 thinking 内容，它会影响上下文窗口判断。
- **插件 API 没有变化**：本变更没有新增 Message Chain 组件、Provider 方法、Adapter 接口或装饰器。插件仍应通过 `AstrMessageEvent`、`Message Chain`、`context.llm_generate(...)` 和 `context.tool_loop_agent(...)` 使用现有能力。
- **上下文压缩行为可能更早出现**：依赖完整历史进行后续处理的 AI 助手或插件，应注意多模态历史会提高估算 token 数，从而影响 `context_limit_reached_strategy` 相关压缩策略的触发时机。
- **可信 token usage 仍覆盖本地估算**：当 `trusted_token_usage` 存在时，估算逻辑不会额外叠加图片、音频或思考内容的估算值。AI 开发者在排查 token 数差异时，应先确认当前路径使用的是 Provider 返回值还是本地估算值。