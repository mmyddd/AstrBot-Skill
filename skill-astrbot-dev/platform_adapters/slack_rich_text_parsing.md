---
title: Slack 富文本段落解析与提及边界保留
type: improvement
status: stable
last_updated: 2026-10-06
related_base: platform_adapters/message_conversion.md
---

## 概述

Slack Platform Adapter 在 `convert_message` 阶段解析 Slack `blocks` 结构时，`_parse_blocks` 现在会把 `rich_text_preformatted`（代码块）、`rich_text_quote`（引用）与 `rich_text_section`（普通富文本段落）一并以相同的内联元素解析流程处理，并在非 `rich_text_section` 的块前后注入换行 `Plain` 组件，以保留块之间的边界。

此前只有 `rich_text_section` 会被识别，代码块与引用中的文本会丢失。该变更使 Slack 消息进入 AstrBot 统一消息模型（Message Chain / `AstrBotMessage.message_str`）时保留代码块和引用文字，同时避免块边界的换行被误插入到 At 组件与其唤醒目标之间。

该变更不新增 Message Chain 组件，也不改变插件侧的 `event.plain_result(...)`、`event.chain_result(...)` 或 `event.message_str` 消费方式。它只影响 Slack Platform Adapter 内部从 Slack `blocks` 到 AstrBot Message Chain 的转换语义。

## 核心行为

核心方法仍然是：

```python
SlackAdapter._parse_blocks(self, blocks: list) -> list
```

稳定行为如下：

1. 遍历 `blocks`，对 `type == "rich_text"` 的块读取其 `elements`。
2. 对 `elements` 中的元素，当 `element.get("type")` 属于以下集合时，统一进入内联元素解析流程：
   - `rich_text_section`
   - `rich_text_preformatted`
   - `rich_text_quote`
3. 记录进入该段落前的组件起始位置：

```python
component_start = len(message_components)
```

4. 解析该段落中的内联 `elements`，复用原有文本、用户提及、链接、表情等解析逻辑，并聚合为 `text_content`。
5. 追加 `Plain(text=text_content)` 的条件为：

```python
if text_content and (
    text_content.strip()
    or element.get("type") != "rich_text_section"
):
```

   这保证了纯空白的 `rich_text_section` 不会被添加，而代码块或引用中的纯空白内容仍会被保留。
6. 对非 `rich_text_section` 的元素（即代码块或引用），在当前段落组件之后追加一个 `Plain(text="\n")`；并且当该段落之前已经存在组件（`component_start` 非 0）时，还会在该段落起始位置插入 `Plain(text="\n")`。

```python
if (
    element.get("type") != "rich_text_section"
    and len(message_components) > component_start
):
    if component_start:
        message_components.insert(component_start, Plain(text="\n"))
    message_components.append(Plain(text="\n"))
```

该设计确保块边界的换行位于代码块/引用的外侧，不会把换行插入到 `At` 组件与其对应的提及目标之间。

## 与消息转换模型的关系

该逻辑位于 Slack Platform Adapter 的消息转换阶段，属于 `convert_message` 内部的 `_parse_blocks` 分支。转换完成后的数据流仍与 `platform_adapters/message_conversion.md` 一致：

1. Slack 平台的原始事件通过 Slack SDK 进入 `SlackAdapter.convert_message(event)`。
2. `_parse_blocks` 将 Slack `blocks` 映射为 AstrBot Message Chain 组件列表（`Plain`、`At` 等）。
3. 转换结果写入 `AstrBotMessage.message` 与 `AstrBotMessage.message_str`。
4. 事件通过 `commit_event(...)` 进入事件队列，插件 Handler 与 LLM Provider 消费统一消息模型。

变更保持 Platform Adapter 的职责边界：Slack 特有的 `rich_text_preformatted` 与 `rich_text_quote` 语义只在适配器层被解释，插件层继续消费统一的 `AstrBotMessage`、`event.message_str` 与 Message Chain。

## 插件开发契约

插件作者在 Slack 平台上处理问题时，应继续依赖统一消息模型：

```python
@filter.command("echo")
async def echo(self, event: AstrMessageEvent):
    yield event.plain_result(event.message_str)
```

当用户在 Slack 中通过引用或代码块发送内容时，`event.message_str` 可能包含由换行分隔的多段文本，例如：

```text
Explain this error
BLOCK_TEST_9527
RuntimeError: unique_test_9527
End
```

插件不应假设 `event.message_str` 一定只包含 `rich_text_section` 的文本，也不应为了提取代码块或引用文字而解析 Slack 原始 `blocks`。如确需访问 Slack 原始 payload，可读取 `event.message_obj.raw_message`，但这属于平台特定逻辑。

## 变更影响分析

- **代码块与引用文字不再丢失**：Slack 的 `rich_text_preformatted` 与 `rich_text_quote` 元素现在会像 `rich_text_section` 一样进入内联元素解析流程。AI 开发者在分析 Slack 消息时，不应假设代码块或引用会被丢弃。
- **块边界以 `Plain("\n")` 表达**：非 `rich_text_section` 的块会在前后追加换行组件，形成可读的多段文本。这属于 Slack 适配器内部的转换语义，不是新的 Message Chain 组件；插件不应主动构造 `rich_text_preformatted` 类组件。
- **提及位置决定换行插入点**：当段落前已存在组件（`component_start` 非 0）时，块前换行会插入到该段落起始处；否则仅在段落末尾追加换行。这避免了块前换行被插入到 `At` 与其提及目标之间，保持唤醒目标识别所需的 “前导 At” 语义。
- **空白内容处理更精确**：`rich_text_section` 的纯空白文本被忽略；但代码块与引用的纯空白文本仍会被保留为 `Plain`。对依赖空行或多行结构的场景，AI 助手应把代码块与引用中的空白视为有效内容。
- **`event.message_str` 可能出现额外换行**：由 Slack 代码块或引用触发的块边界会被映射为 `\n`，因此 `event.message_str` 可能出现以 `\n` 包裹的中间段落。依赖严格字符串匹配的正则或前缀判断，应把该语义变化纳入考虑，或改用 `event.message_obj.message` 消费组件顺序。
- **插件 API 没有变化**：本变更没有新增装饰器、Message Chain 组件、Provider 方法或 Adapter 注册接口。插件仍应通过 `event.message_str`、`event.message_obj.message` 与 `Message Chain` 消费 Slack 消息。
- **不要为 Slack 代码块编写专属解析分支**：Slack 的 `rich_text_preformatted` / `rich_text_quote` 已在 Platform Adapter 层归一化。AI 助手生成跨平台插件代码时，应保持对统一消息模型的依赖，而不是在插件内重新解析 Slack `blocks` 结构。
