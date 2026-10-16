---
title: 会话隔离与 unique_session 行为 (Session Isolation Before Pipeline Registration)
type: improvement
status: stable
last_updated: 2026-10-16
related_base: messages/umo.md
---

## 概述

AstrBot 支持在群聊场景下对会话进行「按用户隔离」，由 `platform_settings.unique_session` 控制。该行为会改写群聊事件的 `event.session_id`（进而改写 `event.unified_msg_origin`），使同一群聊中的不同发送者拥有独立会话上下文。

本次变更把会话隔离的执行位置从 `WakingCheckStage` 内部移动到 `EventBus` 分发阶段。隔离现在发生在 **事件进入 Pipeline 之前**，也就是在事件注册与 Agent 停止查找之前完成。这一时序调整确保整个事件生命周期（唤醒检查、Agent 停止查找、Pipeline 清理）使用的是同一个最终 UMO。

该变更不新增装饰器、Message Chain 组件、Provider 方法或 Adapter 注册接口。它改变的是 `AstrMessageEvent` 的 `session_id` / `unified_msg_origin` 在事件流转早期的确定时机与归属层。

## 核心实现

会话隔离逻辑集中在 `EventBus`：

```python
class EventBus:
    def _apply_session_isolation(
        self,
        event: AstrMessageEvent,
        config: AstrBotConfig,
    ) -> None:
        ...
```

`EventBus.dispatch()` 在取到对应 `PipelineScheduler` 之后、创建 `scheduler.execute(event)` 任务之前调用 `_apply_session_isolation(...)`：

```python
self._apply_session_isolation(event, scheduler.ctx.astrbot_config)
task = asyncio.create_task(scheduler.execute(event))
```

这意味着隔离动作使用的是**路由到该事件时选中的 profile 配置**（即 `scheduler.ctx.astrbot_config`），而不是某个固定全局配置。

原先该逻辑位于 `WakingCheckStage.process()`，并使用 `WakingCheckStage.unique_session` 与 `build_unique_session_id(event)`。此次变更后：

- `astrbot/core/pipeline/waking_check/stage.py` 不再定义 `UNIQUE_SESSION_ID_BUILDERS` 与 `build_unique_session_id`。
- `WakingCheckStage` 不再持有 `self.unique_session`。
- `_session_isolated` 额外信息仍会在每次分发时被显式重置为 `False`，只是重置位置改为 `EventBus._apply_session_isolation(...)`。

## 隔离触发条件

每收到一个事件，`EventBus._apply_session_isolation(...)` 都会先执行：

```python
event.set_extra("_session_isolated", False)
```

仅当以下条件全部满足时，才会真正改写 `session_id`：

1. 选中 profile 的 `platform_settings.unique_session` 为真。
2. `event.message_obj.type == MessageType.GROUP_MESSAGE`（仅群聊消息参与隔离，私聊不受影响）。
3. 该平台在 `UNIQUE_SESSION_ID_BUILDERS` 中存在对应 builder，且 builder 返回非空字符串。

满足时：

```python
event.session_id = sid
event.set_extra("_session_isolated", True)
```

由于 `session_id` 是 `AstrMessageEvent` 的响应式属性（同步到底层 `MessageSession`），改写 `event.session_id` 会立即反映到 `event.unified_msg_origin`。

## 平台级 Session ID 构造契约

`UNIQUE_SESSION_ID_BUILDERS` 定义了各平台在隔离模式下生成的 `session_id` 格式：

| 平台 (`event.get_platform_name()`) | 构造规则 |
| :--- | :--- |
| `aiocqhttp` | `{sender_id}_{group_id}` |
| `slack` | `{sender_id}_{group_id}` |
| `dingtalk` | `{sender_id}` |
| `qq_official` | `{sender_id}_{group_id}` |
| `qq_official_webhook` | `{sender_id}_{group_id}` |
| `lark` | `{sender_id}%{group_id}` |
| `misskey` | `{session_id}_{sender_id}` |
| `matrix` | `{sender_id}_{group_id or session_id}` |

未在此表中的平台（例如 `telegram`）即使开启 `unique_session`，也不会改写群聊 `session_id`，`_session_isolated` 保持为 `False`。

## 与事件流转模型的关系

该变更把会话隔离前移到 EventBus 层，与既有的消息流转模型衔接如下：

1. Platform Adapter 收到平台消息，转换为 `AstrBotMessage`，并通过 `commit_event(event)` 提交到事件队列。
2. `EventBus.dispatch()` 从队列取出事件，并依据**原始 UMO** 通过 `AstrBotConfigManager.get_conf_info(...)` 选择 profile。
3. `EventBus._apply_session_isolation(event, config)` 依据该 profile 的 `unique_session` 设置改写 `event.session_id`，从而得到最终 UMO。
4. 之后才创建 `PipelineScheduler.execute(event)` 任务，事件在最终 UMO 下进入唤醒检查、指令匹配、Agent 停止查找与 Pipeline 清理。

关键点是：配置路由（选择 profile）仍使用**隔离前**的原始 UMO；而事件注册、Agent 停止查找与 Pipeline 后续阶段都使用**隔离后**的最终 UMO。

## 变更影响分析

- **会话隔离时机提前到 EventBus 层**：AI 开发者在理解 AstrBot 事件流时，应把 `unique_session` 视为「事件进入 Pipeline 之前」完成的最终态改写，而不是某个 Pipeline Stage 内部的中间步骤。`WakingCheckStage` 不再负责会话隔离。
- **配置路由与事件注册使用不同阶段的 UMO**：profile 选择使用原始 UMO，Pipeline 及其后的停止查找使用最终 UMO。依赖 UMO 做跨层比对的逻辑必须明确自己处于哪一阶段。
- **`_session_isolated` 仍是可读的稳定标记**：插件仍可通过 `event.get_extra("_session_isolated")` 判断当前事件是否发生了按用户隔离。该额外信息每次分发都会被重置，不会从上一个事件泄漏（测试明确验证了「陈旧标记不会在共享群事件上授予访问权限」这一边界）。
- **平台覆盖范围固定且显式**：只有 `UNIQUE_SESSION_ID_BUILDERS` 中列出的平台在 `unique_session=true` 的群聊中会隔离。插件不应对未列出平台假设隔离生效，也不应自行推导隔离格式，而应读取 `event.session_id` / `event.unified_msg_origin`。
- **私聊不受隔离影响**：隔离条件包含 `MessageType.GROUP_MESSAGE`，私聊消息的 `session_id` 不会被此逻辑改写。
- **仅群聊参与，且按选中的 profile 生效**：同一进程可能存在多个 profile 配置，隔离与否取决于事件被路由到的那个 profile，而不是全局开关。
- **插件 API 没有变化**：本变更没有新增装饰器、Message Chain 组件、Provider 方法、Adapter 接口或 `Context` 方法。插件仍应通过 `event.unified_msg_origin`、`event.session_id`、`AstrMessageEvent` 与 Message Chain 消费会话标识。
- **不要跨阶段缓存 UMO 假定同一阶段语义**：若插件在事件分发早期（如基于原始 UMO 的自定义路由逻辑）缓存了会话标识，需要注意其可能与 Pipeline 阶段的最终 UMO 不同。优先在 Handler 中读取 `event.unified_msg_origin` 获取最终态。
- **调试会话聚合问题时优先检查平台覆盖与开关**：当同一群内不同用户被错误合并或未被隔离时，应先确认平台是否在 `UNIQUE_SESSION_ID_BUILDERS` 中、`unique_session` 是否开启、以及消息类型是否为群聊。
