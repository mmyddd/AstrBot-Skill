---
title: WeCom AI 会话 ID 提取规则
type: improvement
status: stable
last_updated: 2026-06-24
related_base: platform_adapters/adapter_interface.md
---

## 概述

WeCom AI Bot 适配器在从平台原始消息构造 AstrBot 统一消息模型时，`session_id` 的提取规则已按聊天类型区分。

核心方法：

```python
WeComAIAdapter._extract_session_id(message_data: dict[str, Any]) -> str
```

新的规则是：

- 群聊消息：当 `message_data.get("chattype", "single") == "group"` 时，使用 `message_data["chatid"]` 作为会话来源。
- 单聊消息：其他情况使用 `message_data["from"]["userid"]` 作为会话来源。
- 两种路径最终都会调用 `format_session_id("wecomai", value)` 生成适配器内部使用的会话 ID。

## 会话提取逻辑

WeCom AI 原始消息中同时可能存在发送者用户信息和群聊信息。旧逻辑始终从：

```python
message_data.get("from", {}).get("userid", "default_user")
```

提取会话标识，因此群聊中的不同用户消息可能被映射到不同的用户级会话。

新逻辑先读取聊天类型：

```python
chattype = message_data.get("chattype", "single")
```

当聊天类型为群聊时：

```python
chat_id = message_data.get("chatid", "default_group")
return format_session_id("wecomai", chat_id)
```

当聊天类型不是群聊时：

```python
user_id = message_data.get("from", {}).get("userid", "default_user")
return format_session_id("wecomai", user_id)
```

这使 WeCom AI Platform Adapter 的会话隔离更贴近平台语义：群聊以群会话为边界，单聊以用户会话为边界。

## 与 AstrBot 统一消息模型的关系

`session_id` 是 `AstrBotMessage` 的核心字段之一，会进一步参与 `MessageSession` 与 `event.unified_msg_origin` 的构造。插件作者通常不会直接调用 `_extract_session_id`，但会通过以下入口间接受到该规则影响：

```python
event.session_id
event.unified_msg_origin
```

因此，在 WeCom AI Bot 平台上：

- 群聊中的多个成员如果来自同一个 `chatid`，会进入同一个群聊会话上下文。
- 单聊消息仍然按发送者 `userid` 隔离。

该变更保持了 Platform Adapter 的职责边界：平台原始 payload 中的 `chattype`、`chatid`、`from.userid` 由适配器层解释，插件层继续消费统一的 `AstrMessageEvent`、`AstrBotMessage` 与 Message Chain。

## 变更影响分析

- **会话隔离语义变化**：WeCom AI 群聊消息现在按 `chatid` 生成 `session_id`，而不是按发送者 `userid` 生成。依赖 `event.unified_msg_origin`、`event.session_id` 或 ConversationManager 的插件会看到更符合群聊语义的上下文边界。
- **群聊上下文更稳定**：同一 WeCom AI 群聊中的不同用户消息会归入同一群会话，适合群内连续对话、群级状态、群级 Provider 配置或群级会话历史管理。
- **单聊行为保持原语义**：非 `group` 的消息继续使用 `from.userid` 作为会话来源，因此单聊用户之间仍保持隔离。
- **默认值边界明确**：当群聊消息缺少 `chatid` 时，会使用 `default_group`；当单聊消息缺少 `from.userid` 时，会使用 `default_user`。AI 开发者在分析异常会话聚合问题时，应优先检查 WeCom AI 原始 payload 是否包含正确的 `chattype`、`chatid` 或 `from.userid`。
- **插件不应解析平台原始字段来重建会话**：插件应优先使用 `event.unified_msg_origin` 和 `event.session_id`。只有在确实需要 WeCom AI 平台特定逻辑时，才读取 `event.message_obj.raw_message` 中的 `chattype` 或 `chatid`。
- **适配器作者最佳实践**：接入具有群聊和单聊两种会话模型的平台时，`session_id` 应体现真实会话边界。群聊应优先使用群会话 ID，单聊应使用用户 ID，避免把群聊上下文错误拆分为多个用户级上下文。