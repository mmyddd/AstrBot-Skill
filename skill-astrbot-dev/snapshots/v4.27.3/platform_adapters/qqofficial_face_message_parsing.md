---
title: QQ Official 表情消息解析
type: improvement
status: stable
last_updated: 2026-06-24
related_base: platform_adapters/message_conversion.md
---

## 概述

QQ Official 适配器在将平台原始消息转换为 AstrBot 统一消息模型时，会解析 QQ 官方机器人收到的表情或贴纸消息标签，并将其转换为可读文本。

QQ Official 原始表情格式示例：

```text
<faceType=4,faceId="",ext="eyJ0ZXh0IjoiW+a7oeWktOmXruWPt10ifQ==">
```

其中 `ext` 字段包含 base64 编码的 JSON。该 JSON 内的 `text` 字段表示表情描述，例如 `[满头问号]`。适配器会将该标签替换为如下文本：

```text
[表情:[满头问号]]
```

如果 `ext` 字段缺失、base64 解码失败、JSON 解析失败，或 JSON 中没有可用的 `text` 字段，则回退为：

```text
[表情]
```

## 转换位置

该逻辑发生在 QQ Official 适配器的消息转换阶段，即平台原始消息被映射为 `AstrBotMessage` 的过程中。

核心方法：

```python
QQOfficialPlatformAdapter._parse_face_message(content: str) -> str
```

转换后的文本会进入标准消息字段：

- `AstrBotMessage.message_str`：插件常用的纯文本输入。
- `Plain` 消息组件文本：消息链（Message Chain）中的可读文本段。

这意味着插件作者通常无需直接处理 QQ Official 的 `<faceType=...>` 原始标签。对于普通命令、正则匹配、LLM 请求和基于 `event.message_str` 的业务逻辑，表情内容会以 `[表情:描述]` 或 `[表情]` 的形式出现。

## 数据流影响

QQ Official 原始消息进入 AstrBot 后的数据流如下：

1. QQ Official 平台 SDK 提供原始 `message.content`。
2. `QQOfficialPlatformAdapter._parse_from_qqofficial(...)` 执行平台消息转换。
3. 转换过程中调用 `_parse_face_message(content)`。
4. `_parse_face_message` 使用正则匹配 `<faceType=...>` 标签。
5. 对每个匹配到的标签，解析其中的 `ext="..."` 字段。
6. 将 `ext` base64 解码为 JSON，并读取 JSON 的 `text` 字段。
7. 使用 `[表情:<text>]` 替换原始标签；失败时使用 `[表情]`。
8. 转换后的文本写入 `AstrBotMessage.message_str` 或用于构造 `Plain` 消息组件。
9. 后续插件 Handler、事件过滤器、Message Chain 消费者和 LLM Provider 接收的是可读文本，而不是 QQ Official 的原始 XML-like 标签。

该设计保持了 Platform Adapter 的职责边界：平台特有格式在适配器层被归一化，插件层继续消费统一的 `AstrBotMessage` 与 Message Chain。

## 插件开发契约

插件作者在 QQ Official 平台上处理表情消息时，应依赖统一消息模型，而不是解析平台原始标签：

```python
@filter.command("echo")
async def echo(self, event: AstrMessageEvent):
    yield event.plain_result(event.message_str)
```

当用户发送 QQ Official 表情消息时，`event.message_str` 可能包含：

```text
[表情:[满头问号]]
```

或在解析失败时包含：

```text
[表情]
```

如果插件确实需要访问 QQ Official 原始 payload，可继续读取：

```python
event.message_obj.raw_message
```

但这属于平台特定逻辑，插件应优先使用 `event.message_str` 和 `event.message_obj.message` 以保持跨平台兼容。

## 变更影响分析

- **LLM 可读性提升**：LLM Provider 接收到的用户输入不再包含难以理解的 `<faceType=...>` 原始标签，而是包含可读的 `[表情:描述]` 文本，有助于模型理解用户语气和上下文。
- **插件匹配行为变化**：依赖 `event.message_str` 的插件现在会看到 `[表情:...]` 或 `[表情]`。如果插件此前通过正则匹配 `<faceType=` 来识别 QQ Official 表情，该逻辑应改为匹配新的可读文本，或改用 `event.message_obj.raw_message` 处理平台原始数据。
- **Message Chain 语义更统一**：QQ Official 表情不会作为新的独立 `Face` 组件暴露，而是归一化为 `Plain` 文本描述。AI 开发者在 RAG 或代码生成时不应假设 QQ Official 表情会生成 `Face` 组件。
- **解析失败是显式降级**：当 `ext` 字段不可解析时，系统不会丢弃表情，而是保留 `[表情]` 占位。这保证了消息语义中“用户发送了表情”这一事实不会丢失。
- **适配器职责清晰化**：平台特有的表情标签解析被限制在 QQ Official Platform Adapter 内部，不要求插件、Provider 或事件系统理解 QQ Official 的原始消息格式，从而降低跨层耦合。
