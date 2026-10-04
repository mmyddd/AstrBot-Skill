---
title: Telegram Mention 实体 UTF-16 偏移解析
type: improvement
status: stable
last_updated: 2026-10-04
related_base: platform_adapters/message_conversion.md
---

## 概述

Telegram Platform Adapter 在将平台原始消息转换为 AstrBot 统一消息模型时，解析 `mention` 实体（正文与媒体 Caption）的方式已切换为使用 Telegram SDK 提供的实体解析方法，并按 UTF-16 code unit 语义定位实体范围。

旧逻辑直接用 Python 字符串索引对 `entity.offset` / `entity.length` 切片：

```python
name = plain_text[entity.offset + 1 : entity.offset + entity.length]
```

而 Telegram Bot API 中 entity 的 `offset` 与 `length` 以 UTF-16 code unit 计。当消息正文或 Caption 中出现在 `mention` 之前的 emoji（或任何非 BMP 字符）时，Python 字符索引与 UTF-16 偏移不再对齐，会导致 `Comp.At` 的 `name`/`qq` 被错误截取，并可能在移除 bot 自身 mention 时删错字符区间。

该变更不引入新的插件 API、消息组件或装饰器。它只修正 Telegram Platform Adapter 内部将平台实体映射为 `Comp.At` 与 `Plain` 时的偏移语义。

## 核心行为

### 正文 mention（`update.message.text` + `entities`）

适配器现在在构造 `AstrBotMessage` 的过程中处理文本消息，其稳定流程为：

1. 读取 `plain_text = update.message.text`。
2. 若存在 `update.message.entities`，则把 `plain_text` 编码为 UTF-16：

   ```python
   text_utf16 = plain_text.encode("utf-16-le")
   ```

3. 遍历每个 entity，仅处理 `entity.type == "mention"`。
4. 通过 Telegram SDK 解析实体文本，并去掉前导 `@`：

   ```python
   name = update.message.parse_entity(entity)[1:]
   ```

5. 为每个 mention 追加一个 `Comp.At(qq=name, name=name)`。
6. 若该 mention 指向当前 bot（`name.lower() == context.bot.username.lower()`），则把它从 `plain_text` 中移除：
   - 按 UTF-16 码元切片保留区间 `text_utf16[last_end : entity.offset * 2]`，并更新 `last_end = (entity.offset + entity.length) * 2`。
7. 最后把 `text_utf16[last_end:]` 追加到 `text_parts`，再整体 `b"".join(text_parts).decode("utf-16-le")` 得到新的 `plain_text`。

因此，bot 自身 mention 会被从 `message_str` 与 `Plain` 组件中移除，但其它用户的 mention 会以 `Comp.At` 保留，同时不作正文删除。这与升级前的“命中 bot 才删除，否则保留”业务语义一致，但偏移计算已正确对齐 UTF-16。

### 媒体 Caption mention（`caption` + `caption_entities`）

在 `_apply_caption()` 中，处理 `update.message.caption_entities` 的逻辑改为：

```python
name = update.message.parse_caption_entity(entity)[1:]
```

`sdk.parse_caption_entity(entity)` 使用 Telegram SDK 自带的实体解析语义，内部同样按 UTF-16 偏移计算切片。因此对于 `photo`、`document`、`video` 等带 Caption 的媒体消息，mention 解析不再受 Caption 中前置 emoji 的影响。

需要注意：在 Caption 路径中，移除 bot 自身 mention 的正文清洗不再直接作用于 `message_str`。测试 `test_telegram_media_caption_mention_uses_utf16_offsets` 断言 `result.message_str == caption`（即 Caption 原文保留），同时 `Comp.At` 组件仍按解析结果生成。开发者若依赖 Caption 的纯文本字段做匹配，应理解 JSON-Caption 中 bot mention 前缀可能仍出现在 `message_str` 中，业务侧应以消息链中的 `Comp.At` 为准识别提及对象。

### 保持不变的业务契约

- 消息链仍只使用标准消息组件（`Comp.At`、`Comp.Plain`），不新增 `Mention`/`Entity` 类型。
- `AstrBotMessage.message` 仍按顺序追加 `Comp.At` 与 `Comp.Plain`。
- `AstrBotMessage.message_str` 仍为正文的纯文本汇总。
- `session_id`、`group`、`sender`、`raw_message`、`timestamp` 等字段的填充逻辑不因该变更而改变。
- bot 自身 mention 的“从正文移除”的语义在正文路径上保持不变，只是偏移计算更可靠。

## 与统一消息模型的关系

该变更位于 Telegram Platform Adapter 的消息转换阶段，属于 `platform_adapters/message_conversion.md` 中所述 `convert_message` 的实体归一化职责范围。Telegram 平台特有的 entity（`mention`）仍被适配器层解析，并映射为统一消息模型中的 `Comp.At`；插件层继续消费 `AstrBotMessage`、`Message Chain`、`AstrMessageEvent`。

数据流如下：

1. Telegram SDK 提供 `update.message.text` / `caption` 以及对应的 `entities` / `caption_entities`。
2. Telegram Adapter 遍历 `mention` 实体，使用 `parse_entity(...)` / `parse_caption_entity(...)` 得到规范化 mention 文本。
3. 去掉前导 `@` 后构造 `Comp.At(qq=name, name=name)`。
4. 正文路径使用 UTF-16 码元切片，将 bot 自身 mention 从 `plain_text` 中移除。
5. 改写后的 `plain_text` 被写入 `AstrBotMessage.message_str` 并作为 `Comp.Plain` 追加到消息链。
6. 后续 Event Loop、Handler、LLM Provider、Message Chain 消费者只看到归一化后的 `Comp.At` 与 `Plain`，不接触 Telegram entity 偏移。

## 变更影响分析

- **提及解析不再对 emoji 敏感**：正文与 Caption 中出现在 `@bot` 之前的 emoji（或其它非 BMP 字符）不会再导致 mention 名称截取错误。依赖 Bot 识别“是否被 At 到自己”的插件与 Provider 触发逻辑因此更稳定。
- **多 mention 场景的正文切分更准确**：测试覆盖了 `"😀 @test_bot @alice @test_bot hello"` 这类同时包含多个 mention 的场景，`Comp.At` 列表与 `message_str` 都按原顺序保留。插件在对 `event.message_obj.message` 做遍历以识别 `At` 目标时，不应假设 `At` 一定紧邻 `Plain`；此处仍按实体出现顺序追加到消息链。
- **Caption 路径不保证 `message_str` 中被清除 bot mention**：`test_telegram_media_caption_mention_uses_utf16_offsets` 断言 `result.message_str == caption`。AI 助手生成用于 Telegram 媒体消息的业务匹配逻辑时，应优先使用消息链中的 `Comp.At` 判断提及对象，而不是只依赖 `event.message_str`。
- **插件侧 API 没有变化**：本变更未新增装饰器、`Context` 方法、`Message Chain` 组件、Provider 接口或 Adapter 注册接口。插件仍应使用 `event.message_str`、`event.message_obj.message` 与 `Comp.At`、`Comp.Plain` 等既有契约。
- **不要按 Python 字符索引直接解析 Telegram entity**：如果插件需要访问 `event.message_obj.raw_message` 中的 Telegram `entities` 并自行切分 `text`，必须意识到 `offset`/`length` 以 UTF-16 码元计，不能直接 Python 字符串下标切片。优先复用 Telegram SDK 的 `parse_entity` / `parse_caption_entity`。
- **适配器职责边界更清晰**：把 Telegram 实体解析完全下沉到 SDK 方法，减少适配器内手写偏移切片的概率；同时保留“bot 自身 mention 从正文删除、其他 mention 以 `Comp.At` 保留”的旧语义，从而避免影响现有命令匹配与 At 触发逻辑。
- **测试面覆盖要点**：新增测试以 `Message.de_json(...) + update.message.parse_entity = sdk_message.parse_entity` 的方式验证偏移语义，并覆盖 `photo`、`document`、`video` 三类带 Caption 的媒体消息。AI 助手在生成涉及 Telegram Adapter 的回归测试时，应沿用这种“SDK 解析器 + UTF-16 实体”的组合来构造用例。
- **`Comp.At` 的 `qq` 含义**：此处 `qq` 被设置为 `name`（去掉前导 `@` 后的 username 字符串），与 Telegram 用户 ID 并非同一概念。这是 Telegram 平台适配的既有约定，此次变更未改变该约定，仅在解析来源上改用了 SDK 方法。
