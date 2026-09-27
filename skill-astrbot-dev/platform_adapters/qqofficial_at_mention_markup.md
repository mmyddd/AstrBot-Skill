---
title: QQ Official @ 提及序列化与入站清理
type: feature
status: stable
last_updated: 2026-09-27
related_base: platform_adapters/message_conversion.md
---

## 概述

QQ Official Platform Adapter 的 `At` 消息组件处理和入站提及清理逻辑已按 QQ 官方协议对齐。插件侧继续使用标准的 `At(qq="user_openid")` 消息组件与 Message Chain 表达提及，Platform Adapter 在出站序列化阶段将其映射为 QQ Official 原生提及标签 `<qqbot-at-user id="<openid>" />`；在入站转换阶段，适配器统一剥离当前机器人被提及的多种标记格式，使 `event.message_str` 中不再残留平台原始标签。

该变更的核心契约是：`At` 组件在 QQ Official 平台上是端到端可用的，插件不需要自行拼接 `<qqbot-at-user ... />` 或 `<@openid>` 标记。

## 出站序列化：`_parse_to_qqofficial`

出站路径继续由以下静态方法负责，将 `MessageChain` 转为 QQ Official 文本载荷：

```python
async def _parse_to_qqofficial(message: MessageChain)
```

新增行为：遍历 `message.chain` 时，若遇到 `At` 组件，则将其序列化为 QQ Official 提及标签，并追加到 `plain_text`：

```python
elif isinstance(i, At):
    mention_id = str(i.qq) if i.qq else ""
    if mention_id and mention_id != "all":
        plain_text += f'<qqbot-at-user id="{mention_id}" />'
```

稳定契约：

- `At.qq` 为有效非空值且不等于 `all` 时，生成 `<qqbot-at-user id="<qq>" />`。
- `At.qq` 为空或 `None` 时被忽略，不产生任何标签，避免产生无 id 的非法提及。
- `At.qq == "all"`（AtAll 语义）不生成 QQ Official 提及标签，即 QQ Official 当前不支持 `@全体成员` 的此序列化路径。
- 序列化保持组件在 Message Chain 中的原始顺序，`At` 与相邻 `Plain` 文本按顺序拼接为同一段 `plain_text`。

## 入站清理：`_strip_bot_mention_markup`

入站路径新增静态方法，用于统一清理入站内容中针对当前机器人的提及标记：

```python
@staticmethod
def _strip_bot_mention_markup(content: str | None, mention_id: str) -> str
```

该方法使用正则同时匹配两类标记：

```python
markup_pattern = (
    rf'(?:<qqbot-at-user\s+id="{escaped_id}"\s*/>|<@!?{escaped_id}>)'
)
```

- 当前（新版）标记：`<qqbot-at-user id="<openid>" />`。
- 历史标记：`<@<openid>>` 与 `<@!<openid>>`。

清理规则：

- 仅移除目标 `mention_id` 对应的标记，其他用户的 `<@...>` 提及保持不变。
- 匹配到标记时，会一并吞掉标记前后的空格或制表符，并替换为单个空格，以避免 `before<@bot>after` 变成 `beforeafter`，而应变为 `before after`，保留词边界。
- 对 `mention_id` 进行 `re.escape`，因此形如 `bot.123` 的 openid 也能被正确匹配。
- `content` 为 `None` 时视作空字符串。

调用点为 `_parse_from_qqofficial`：群聊分支对每个 `bot_mention_ids` 条目依次调用该方法；非群聊分支则使用 `str(abm.self_id)` 作为 `mention_id` 调用。清理后的结果再交给 `_parse_face_message` 做表情解析，然后写入 `abm.message_str`。

这替代了此前仅对 `<@id>` 与 `<@!id>` 做简单字符串 `replace` 的实现，且新增对 `<qqbot-at-user ... />` 的识别。

## 与统一消息模型的关系

该变更没有新增消息组件。插件侧仍使用标准 `At`：

```python
import astrbot.api.message_components as Comp
yield event.chain_result([Comp.At(qq="member-1"), Comp.Plain(" hello")])
```

平台差异仅在 Platform Adapter 层被解释：

- 出站：`At` → QQ Official `<qqbot-at-user id="..." />`。
- 入站：QQ Official 的提及标记在消息转换阶段被剥离，不进入 `event.message_str`。

该设计与「平台特有格式在适配器层归一化，插件层消费统一 `AstrBotMessage` 与 Message Chain」的原则一致。

## 消息发送路径

`send_by_session` 在 QQ Official 群聊会话下，会因链中包含 `At` 组件而使用 QQ Official 文本链（`post_group_message`）承载提及标记，且该发送路径不携带 `markdown` 参数：

```python
await adapter.send_by_session(
    MessageSession("qq_official", MessageType.GROUP_MESSAGE, "group-1"),
    chain,
)
kwargs = adapter.client.api.post_group_message.await_args.kwargs
kwargs["content"] == '<qqbot-at-user id="member-1" /> hello'
```

稳定行为为：`At` 组件经由文本链序列化后写入 `content` 字段，接收方看到的是原生 QQ Official 提及，而非字面量文本。

## 变更影响分析

- **`At` 组件在 QQ Official 上成为可用的跨层契约**：插件可直接构造 `At` 表达提及，无需自行拼接平台标记。AI 助手在生成针对 QQ Official 的插件代码时，应使用 `Comp.At(qq=...)` 而不是硬编码 `<qqbot-at-user ... />`。
- **`AtAll` 不被序列化**：`At.qq == "all"` 会被 `_parse_to_qqofficial` 显式跳过。不要假设 QQ Official 支持 `@全体成员` 的该编码路径；如需要 `@全体`，目前不存在对应的消息组件映射。
- **空 `At.qq` 会被静默丢弃**：`At(qq=None)` 或 `At(qq="")` 不会产生提及标记。生成 `At` 组件时应保证 `qq` 为有效非空 openid。
- **入站文本已剥离机器人提及标记**：依赖 `event.message_str` 的插件在 QQ Official 群里不会再看到 `<@...>` 或 `<qqbot-at-user ... />` 残留文本。此前依赖 `<@` 前缀做匹配的插件应改为使用平台判定（如 `event.is_at_or_wake_command` / `event.is_wake`）或直接读取 `event.message_obj.raw_message`。
- **其他用户的提及保留在文本中**：剥离仅针对当前机器人 openid。若消息中同时提及了其他用户与机器人，其他用户的 `<@...>` 文本不会被清理，插件在其上做正则匹配时仍可能看到这些标记。
- **清理会保留词边界**：`_strip_bot_mention_markup` 将标记替换为单个空格，因此 `before<@bot>after` 会变为 `before after` 而不是被粘连。AI 助手在推理用户实际输入语义时，应把该空格视为清理产物而非用户输入。
- **清理对 openid 中的正则元字符安全**：`re.escape` 保证含 `.` 等字符的 openid 仍能被精确匹配，插件不应假设 openid 为纯字母数字。
- **入站消息 UTF-8 文本的原始标签仅供参考**：入站并未把 QQ Official 提及转换为独立的 `At` 组件，而是作为文本清理。AI 助手在分析 `event.message_obj.message` 时，不应假设 QQ Official 入站提及会以 `At` 组件形式出现。
- **发送路径由 Adapter 决定**：链中出现 `At` 时由适配器选择文本链发送，插件无需为 QQ Official 群聊单独指定发送通道，也不应绕过 `event.send` / `context.send_message` 直接调用 QQ SDK。
