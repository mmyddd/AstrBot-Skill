---
title: WeCom AI Bot 空消息发送 No-op 行为
type: improvement
status: stable
last_updated: 2026-06-24
related_base: platform_adapters/adapter_interface.md
---

## 概述

WeCom AI Bot 平台事件在处理 `event.send(None)` 时，现在会直接返回，不再继续执行平台发送逻辑。

核心方法：

```python
WecomAIBotMessageEvent.send(self, message: MessageChain | None) -> None
```

稳定行为为：当 `message is None` 时，`WecomAIBotMessageEvent.send(...)` 立即 `return`。这使 WeCom AI Bot 长连接场景中的空占位发送不会覆盖或清空已有输出。

该变更不新增 Message Chain 组件，不改变 `AstrMessageEvent` 的通用发送接口，也不要求插件作者为 WeCom AI Bot 单独构造平台特定消息。它只明确了 WeCom AI Bot Platform Adapter 对空消息占位的处理边界。

## 数据流位置

Pipeline 调度器在事件处理阶段结束后，会对特定事件类型执行一次空消息发送：

```python
if isinstance(event, WebChatMessageEvent | WecomAIBotMessageEvent):
    await event.send(None)
```

该调用用于让后续处理流程获得一次统一的发送收尾机会。对于 WeCom AI Bot，事件对象进入平台发送层后会先检查参数：

```python
if message is None:
    return
```

因此，完整数据流为：

1. Platform Adapter 将平台原始消息转换为 `AstrBotMessage`。
2. 事件进入 Pipeline Scheduler。
3. 插件 Handler、LLM Provider 或其他流程可能已经产生真实回复。
4. Pipeline Scheduler 对 `WecomAIBotMessageEvent` 调用 `event.send(None)`。
5. `WecomAIBotMessageEvent.send(...)` 识别到 `message is None`，立即返回。
6. 空占位发送不会进入 WeCom AI Bot 平台的实际回复构造逻辑。

该设计把“Pipeline 需要发送收尾占位”和“WeCom AI Bot 平台不能把空占位当作真实回复”两个职责分离：调度层仍可保持统一流程，适配器层负责解释平台可接受的发送语义。

## 插件开发契约

插件作者在 WeCom AI Bot 平台上仍应使用标准 `AstrMessageEvent` 与 Message Chain 发送消息：

```python
@filter.command("ping")
async def ping(self, event: AstrMessageEvent):
    yield event.plain_result("pong")
```

或在必须主动发送的 hook/session 场景中使用：

```python
await event.send(event.plain_result("pong"))
```

插件不应依赖 `event.send(None)` 来表达业务回复。对于 WeCom AI Bot，`None` 明确表示空占位，适配器会忽略它。

如果插件需要发送空文本或占位提示，应显式构造 `Plain` 或 `MessageChain`，而不是传入 `None`：

```python
from astrbot.api.event import MessageChain

await event.send(MessageChain().message("处理完成"))
```

## 与统一消息模型的关系

此变更不改变以下统一模型契约：

- `AstrBotMessage` 仍由 Platform Adapter 构造。
- `MessageChain` 仍是插件与平台发送层之间的标准消息结构。
- `AstrMessageEvent.send(...)` 仍是事件回复入口。
- `event.unified_msg_origin`、`event.session_id`、Provider 选择和 ConversationManager 行为不受此变更影响。

差异仅发生在 WeCom AI Bot 事件发送实现中：`None` 不会被解释为可发送的消息内容。

## 变更影响分析

- **WeCom AI Bot 长连接回复更稳定**：Pipeline Scheduler 触发的 `event.send(None)` 不会再进入 WeCom AI Bot 的实际发送逻辑，从而避免空占位回复影响已有输出。
- **AI 开发者不应把 `None` 当作跨平台消息内容**：在 AstrBot 的 Message Chain 语义中，业务消息应显式使用 `MessageChain` 或消息组件表达。`None` 在 WeCom AI Bot 中是内部空占位，不是可移植的插件发送内容。
- **适配器作者可参考该边界处理**：如果某个平台事件会被调度层传入空占位发送，适配器的 `send` 实现应明确区分“无消息”与“发送空文本”，避免把内部控制信号映射为平台真实消息。
- **Pipeline 与 Adapter 职责更清晰**：调度器负责统一事件生命周期收尾；WeCom AI Bot Adapter 负责决定空消息是否能被平台消费。该分层降低了调度逻辑与平台协议之间的耦合。
- **插件调试时应检查真实 Message Chain**：如果 WeCom AI Bot 插件没有产生回复，开发者应确认 Handler 是否 yield/发送了实际 `MessageChain` 或 `MessageEventResult`，而不是关注 Pipeline 的 `send(None)` 收尾调用。
- **不新增对外 API**：本变更没有新增装饰器、Provider 方法、Message Chain 组件或 Adapter 注册接口。它是对现有 WeCom AI Bot 事件发送行为的稳定性修正。