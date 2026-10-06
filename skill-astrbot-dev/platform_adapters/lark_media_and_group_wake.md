---
title: Lark 媒体接收与群聊唤醒语义
类型: improvement
状态: stable
最后更新: 2026-10-06
相关基础文档: platform_adapters/adapter_interface.md
---

## 概述

Lark (飞书) Platform Adapter 的对外消息能力自 v4.16.0 起从“单向发送型适配器”升级为“双向多媒体适配器”。该变更包含三条互相独立的契约：

1. **接收侧媒体能力扩展**：Lark 适配器现在可以接收语音、视频与文件消息，并将其纳入 AstrBot 统一消息模型（Message Chain / `AstrBotMessage`）。发送侧仍支持语音、视频、文件。这意味着 `event.plain_result(...)`、`event.chain_result([...])`、`AstrMessageEvent` 与 LLM Provider 在 Lark 平台上能消费到更完整的消息内容。
2. **引用（回复）消息解析**：适配器在接收引用消息时，会解析被引用消息中的文本、图片、视频、文件内容。这属于消息转换阶段（`convert_message` 语义）的一部分，影响 `AstrBotMessage.message` 与 `event.message_str` 的内容。
3. **群聊投递与唤醒的解耦**：群聊中“飞书是否把事件推送给 AstrBot”（投递层，依赖 `im:message.group_msg` 权限）与“AstrBot 收到消息后是否响应”（唤醒层）是两个独立概念。AstrBot 在群聊中响应以下消息：`@` 机器人、`@` 全体成员（除非禁用）、引用机器人消息、以及以唤醒前缀（例如 `/`）开头的消息。

该变更不新增 Message Chain 组件，也不改变插件侧构造与发送消息的方式；平台差异仍被收敛在 Lark Platform Adapter 内部。

## 版本与配置契约

- 该能力的基准版本为 **AstrBot v4.16.0**（此前项目内将 Lark 接入说明标注为 v4.15.0）。
- 扫码一键创建的版本要求标注为 `>= v4.25.0`。
- Webhook 加密相关配置的对外字段名被明确为：
  - `Encrypt Key` 对应配置项 `lark_encrypt_key`
  - `Verification Token` 对应配置项 `lark_verification_token`
- 国际版飞书与企业自部署不再强制走 Webhook：`socket` 长连接在国内版与国际版飞书上均可用；`webhook` 仅在企业自部署且长连接不可用时作为回退。
- 推荐权限集合收敛为：
  - `im:message`（覆盖发送与接收消息）
  - `im:message:send_as_bot`
  - `im:resource`（覆盖图片、文件等资源的上传与下载）
  - `contact:contact.base:readonly`、`contact:user.base:readonly`
  - 群聊场景额外需要 `im:message.group_at_msg:readonly`、`im:message.group_msg`
  - 流式输出额外需要 `cardkit:card:write`
- 旧权限名 `im:message:send` 与 `im:resource:upload` 已不再作为新应用推荐项；它们分别被 `im:message` 与 `im:resource` 覆盖。

## 与统一消息模型的关系

Lark 适配器继续遵循 `platform_adapters/message_conversion.md` 定义的转换契约：把平台原始事件映射为 `AstrBotMessage`，再由事件系统封装为 `AstrMessageEvent` 提交到事件队列。

- 接收语音：适配器在消息转换阶段把 Lark 语音内容归一化为 AstrBot 可消费的媒体内容（语音在接收时转为 `wav`）。
- 接收视频：发送方向会被规范化为 `mp4`（“Video | 发送时转为 mp4”）。视频在接收与发送两个方向均受支持。
- 接收文件：Lark 文件消息现在会被纳入统一消息模型，`File` 组件语义在该平台两个方向均为受支持。
- 引用消息：被引用内容中的文本、图片、视频、文件会被解析，进入 `AstrBotMessage.message` 与 `event.message_str`。

插件作者仍应消费 `event.message_str`、`event.message_obj.message`、`event.unified_msg_origin` 等统一模型入口，不需要解析 Lark 原始 payload。Lark 特有的媒体格式转换（例如语音 `wav` / 音视频 `mp4` / 语音发送 `opus`）由 Platform Adapter 层负责。

## 群聊投递与唤醒双层模型

该文档新增一个重要的语义分层：**投递层（delivery）** 与 **唤醒层（wake）** 不是同一件事。

- **投递层**：飞书平台侧决定把哪些群聊事件推送给 AstrBot。默认仅推送 `@` 机器人的消息。若需接收群聊所有消息，需要 `im:message.group_msg` 权限。
- **唤醒层**：AstrBot 侧在收到消息后决定是否响应。除了 `@` 机器人之外，还包括 `@` 全体成员（除非禁用，对应 `ignore_at_all` 类开关语义）、引用机器人消息、以及以唤醒前缀（例如 `/`）开头的消息。

因此，让机器人在群聊中响应某条消息，与让飞书把该消息推送给 AstrBot，是两个独立需要分别满足的条件。

## 插件开发契约

插件作者在 Lark 平台上应继续依赖统一消息模型：

```python
@filter.command("echo")
async def echo(self, event: AstrMessageEvent):
    yield event.plain_result(event.message_str)
```

收到语音、视频或文件消息时，`event.message_obj.message` 中会出现对应的媒体组件，插件无需为 Lark 编写专属格式分支。收到引用消息时，引用内容中的文本/图片/视频/文件会被解析进统一消息模型，插件可以从 `event.message_str` 或 `event.message_obj.message` 中读取。

如确需访问 Lark 原始 payload，可读取 `event.message_obj.raw_message`，但这属于平台特定逻辑，插件应优先使用统一字段。

## 变更影响分析

- **Lark 从发送型适配器升级为双向多媒体适配器**：AI 开发者在生成 Lark 相关插件或分析 Lark 消息流时，不应再假定 Lark 只支持发送语音/视频/文件而无法接收。媒体在两个方向均为受支持能力。
- **语音在接收与发送方向做不同规范化**：接收时语音转 `wav`，发送时语音转 `opus`。视频发送时转 `mp4`。这些格式归一化属于 Platform Adapter 层，插件不应在业务逻辑中重复实现 Lark 专属格式转换，也不应假设媒体文件扩展名与平台传输格式一致。
- **引用消息内容会进入统一消息模型**：引用消息中的文本、图片、视频、文件会被解析进 `AstrBotMessage.message` 与 `event.message_str`。AI 助手在分析 Lark 输入时，应把引用内容视作有效上下文，而不是只处理用户新输入的外层文本。
- **群聊投递与唤醒必须分层理解**：默认情况下飞书只把 `@` 机器人的群聊消息推送给 AstrBot；`@` 全体成员、引用机器人、唤醒前缀等唤醒条件只有在消息已被投递的前提下才会生效。AI 助手在排查“机器人在群里不响应”问题时，应同时检查：(a) 是否因缺少 `im:message.group_msg` 导致消息未被投递，(b) 消息是否满足唤醒层条件（`@` 机器人、`@` 全体、引用机器人、唤醒前缀）。
- **权限名称契约已更新**：新应用应使用 `im:message` 与 `im:resource`，而不是 `im:message:send` 与 `im:resource:upload`。旧的权限名不应再出现在新插件或文档建议里。AI 助手生成 Lark 接入步骤时，应使用新的权限集合。
- **配置字段名契约已明确**：加密相关配置为 `lark_encrypt_key` 与 `lark_verification_token`。AI 助手在生成 Lark Webhook 相关配置片段时，应使用这两个字段名，而不是笼统的 `encrypt_key` / `verification_token`。
- **长连接不再绑定国内版**：`socket` 长连接对国内版与国际版飞书均可用。Webhook 是回退方案，而不是国际版的必备路径。AI 助手在生成部署指引时，不应再把“国际版需要 Webhook”作为硬性建议。
- **插件 API 没有变化**：本变更没有新增 Lark 专属装饰器、Message Chain 组件、Provider 方法或 Adapter 注册接口。它改变的是 Lark Platform Adapter 在消息转换、媒体格式规范化、群聊投递语义和权限配置上的对外契约。
- **平台兼容性矩阵需更新**：`messages/components.md` 等相关平台能力表中，Lark (飞书) 的语音、视频、文件接收能力应标记为支持，与其发送能力保持一致。
