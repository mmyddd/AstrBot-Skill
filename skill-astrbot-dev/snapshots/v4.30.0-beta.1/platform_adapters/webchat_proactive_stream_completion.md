---
title: WebChat 主动消息持久化与流式完成事件
type: improvement
status: stable
last_updated: 2026-06-24
related_base: platform_adapters/adapter_interface.md
---

## 概述

WebChat Platform Adapter 在通过 `send_by_session(...)` 处理主动消息时，调整了消息投递、持久化和流式完成事件的行为。

该变更影响插件通过统一发送入口向 WebChat 会话主动发送 Message Chain 的场景，例如：

```python
await self.context.send_message(event.unified_msg_origin, message_chain)
```

核心变化是：当 WebChat 会话没有可消费该主动消息的活跃流式请求时，适配器会直接持久化主动消息并返回，避免创建不会被消费的临时队列；当存在活跃流式请求时，适配器会把消息发送到这些请求队列，并显式发送一次 `complete` 事件，帮助 WebChat 侧把该 Message Chain 作为一次完整输出收尾。

## 核心行为

WebChat 主动发送入口仍是平台适配器的标准方法：

```python
WebChatAdapter.send_by_session(session, message_chain)
```

适配器会根据当前会话的 `conversation_id` 查询后端请求队列：

```python
active_request_ids = self._webchat_queue_mgr.list_back_request_ids(conversation_id)
```

随后将请求 ID 分为两类：

- 普通流式请求：`not req_id.startswith("ws_sub_")`
- 被动订阅队列：`req_id.startswith("ws_sub_")`

新的目标选择规则为：

1. 优先向普通流式请求发送主动消息。
2. 如果没有普通流式请求，则退回到当前所有活跃请求。
3. 如果完全没有活跃请求，则直接持久化主动消息，不再创建临时 `active_...` 队列。

当没有可投递的活跃请求时，稳定行为为：

```python
await self._save_proactive_message(conversation_id, message_chain)
await super().send_by_session(session, message_chain)
return
```

这表示 WebChat 主动消息在没有前端流式消费者时仍会被保存到会话历史中，同时不会产生无人消费的队列消息。

## 流式完成事件

`WebChatMessageEvent._send(...)` 新增内部参数：

```python
emit_complete: bool = False
```

当 WebChat Adapter 向活跃请求投递主动消息时，会调用：

```python
await WebChatMessageEvent._send(
    request_id,
    message_chain,
    session.session_id,
    streaming=True,
    emit_complete=True,
)
```

当 `emit_complete=True` 时，`_send(...)` 会在完成 Message Chain 各组件入队后，再向 WebChat 后端队列写入一条完成事件：

```python
{
    "type": "complete",
    "data": data,
    "streaming": streaming,
    "chain_type": message.type,
    "message_id": message_id,
}
```

其中：

- `type="complete"` 表示本次消息输出完成。
- `data` 是 `_send(...)` 对 Message Chain 发送过程生成的数据。
- `streaming` 保留当前发送是否属于流式路径。
- `chain_type` 来自 `message.type`。
- `message_id` 对应本次投递使用的请求或消息标识。

该行为使 WebChat 的主动消息发送更接近普通流式回复的数据流：消息内容先进入后端队列，随后再通过 `complete` 事件标记一次完整输出结束。

## 被动订阅队列的持久化兜底

如果当前会话只有 `ws_sub_` 开头的被动订阅队列，而没有普通流式请求，WebChat Adapter 仍会执行主动消息保存作为兜底：

```python
if not stream_request_ids:
    await self._save_proactive_message(conversation_id, message_chain)
```

原因是被动订阅队列并不等价于普通流式请求，它们不一定绑定到常规的流式持久化路径。通过额外保存主动消息，系统可以避免 WebChat 会话中出现“消息被投递给订阅队列但未进入持久历史”的不一致状态。

## 与统一消息模型的关系

该变更不新增 Message Chain 组件，也不改变插件构造消息的方式。插件仍应使用标准消息组件与统一发送入口：

```python
from astrbot.api.event import MessageChain

chain = MessageChain().message("任务已完成")
await self.context.send_message(event.unified_msg_origin, chain)
```

变化发生在 WebChat Platform Adapter 内部的发送阶段：适配器根据 WebChat 会话当前是否存在活跃请求，决定是直接持久化、向流式队列投递，还是在只有被动订阅队列时额外保存兜底。

因此，插件作者不需要为 WebChat 单独创建队列、生成请求 ID，或手动发送完成事件。主动发送仍应通过 `Context.send_message(...)` 或平台标准发送路径完成。

## 数据流影响

WebChat 主动消息的新数据流可以概括为：

1. 插件或核心逻辑通过 `Context.send_message(...)` 向 WebChat UMO 发送 Message Chain。
2. `WebChatAdapter.send_by_session(...)` 根据 `session.session_id` 解析会话并查询活跃请求队列。
3. 如果存在普通流式请求，消息会投递到这些请求，并附带 `complete` 事件。
4. 如果只有被动订阅队列，消息会投递到这些队列，同时执行主动消息持久化兜底。
5. 如果没有任何活跃请求，消息直接持久化，并调用平台基类发送收尾逻辑后返回。

该设计降低了 WebChat 主动消息对前端实时连接状态的依赖：即使当前没有活跃流式消费者，消息仍可进入会话持久历史；如果存在流式消费者，则可以通过 `complete` 事件获得明确的输出边界。

## 变更影响分析

- **WebChat 主动消息不再依赖临时队列兜底**：当没有活跃请求时，适配器不再创建未被消费的 `active_...` 队列，而是直接保存主动消息。AI 开发者在理解 WebChat 主动发送时，应把“无活跃请求”视为持久化路径，而不是排队等待路径。
- **流式消费者会收到完成边界**：向活跃流式请求投递主动消息时，`WebChatMessageEvent._send(...)` 会额外写入 `complete` 事件。处理 WebChat 后端队列的逻辑可以依赖该事件判断一次主动 Message Chain 输出已经结束。
- **被动订阅队列不是普通流式请求**：`ws_sub_` 开头的请求 ID 会被视为订阅类队列。当会话只有这类队列时，适配器会保存主动消息作为兜底，避免订阅投递和历史持久化之间出现断层。
- **插件 API 没有变化**：本变更没有新增装饰器、Message Chain 组件、Provider 方法或 Adapter 注册接口。插件仍应通过 `Context.send_message(...)`、`event.send(...)` 或标准 Message Chain 结果构造发送消息。
- **不要手动构造 WebChat 请求 ID**：主动消息的请求选择与完成事件由 WebChat Adapter 管理。插件作者不应依赖 `ws_sub_`、流式请求 ID 或内部队列命名来实现业务逻辑。
- **WebChat 会话历史更稳定**：没有实时消费者或只有被动订阅消费者时，主动消息仍会被保存到对应 conversation。这使 AI 助手在分析 WebChat 历史时，可以预期主动发送的 Message Chain 更可靠地进入持久会话记录。
- **完成事件不代表新的消息组件**：`complete` 是 WebChat 队列协议中的事件类型，不是 AstrBot 统一消息模型中的 Message Component。AI 开发者不应在插件 Message Chain 中构造 `complete` 组件。