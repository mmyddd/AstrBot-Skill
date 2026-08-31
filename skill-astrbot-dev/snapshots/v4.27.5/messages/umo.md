---
title: UMO 配置路由的冒号兼容
type: improvement
status: stable
last_updated: 2026-06-24
related_base: messages/umo.md
---

## 概述

UMO（Unified Message Origin）的标准格式仍然是：

```text
platform_id:message_type:session_id
```

本次变更明确了 `umop_config_router` 在处理 UMO 路由规则时的解析契约：解析逻辑只按前两个 `:` 分隔 UMO，第三段整体作为 `session_id` 保留。因此，当平台适配器生成的 `session_id` 本身包含 `:` 时，UMO 配置路由仍可正确识别为三个逻辑部分。

该变更不改变插件侧获取 UMO 的入口。插件仍应通过 `event.unified_msg_origin` 获取当前会话的统一消息源，并在主动发送、会话配置、Provider 选择或上下文隔离相关逻辑中继续使用该值。

## 核心行为

`umop_config_router` 新增内部解析函数：

```python
def _split_umo(umo: str) -> tuple[str, str, str] | None
```

解析规则为：

1. 输入必须是 `str`。
2. 使用 `umo.split(":", 2)` 进行拆分。
3. 只有拆分结果恰好包含三个逻辑部分时才视为合法。
4. 第三个部分作为完整 `session_id` 返回，即使其中继续包含 `:` 也不会再次拆分。

例如：

```text
platform:group:room:thread:123
```

会被解析为：

```text
platform_id  = "platform"
message_type = "group"
session_id   = "room:thread:123"
```

这与旧的全量 `split(":")` 行为不同。旧逻辑会把 `session_id` 内部的冒号也拆开，导致整体不再满足三个字段的校验要求。

## 路由匹配与校验影响

该解析逻辑被用于 UMO 配置路由的匹配和维护方法：

- `_is_umo_match(p1: str, p2: str) -> bool`
- `update_routing_data(new_routing: dict[str, str]) -> None`
- `update_route(umo: str, conf_id: str) -> None`
- `delete_route(umo: str) -> None`

上述方法现在共享同一套 UMO 合法性判断：只要求 UMO 具有 `platform_id`、`message_type`、`session_id` 三个逻辑部分，不再因为 `session_id` 内部包含 `:` 而判定为非法。

路由匹配仍保持原有语义：每个逻辑部分支持空字符串或通配符匹配。匹配过程对三个逻辑字段逐项比较，而不是对原始字符串进行整体比较。

## 与 Message Chain / Adapter / Provider 的关系

此变更位于 UMO 配置路由层，不改变 Message Chain、消息组件、Platform Adapter 消息转换或 Provider 调用接口。

Platform Adapter 仍负责生成 `AstrBotMessage.session_id`，事件系统仍通过 `MessageSession` 暴露 `event.unified_msg_origin`。变化点仅在于：当后续配置路由逻辑消费 UMO 字符串时，会以“前两段固定、第三段整体保留”的方式解析，从而降低平台会话 ID 格式与 AstrBot 配置路由之间的耦合。

对于插件作者，这意味着插件不需要手动清洗 `event.unified_msg_origin` 中的冒号，也不应假设 `session_id` 永远不包含 `:`。如果插件需要拆分 UMO，应遵循同样的三段逻辑：只拆分前两个冒号，第三段整体视为会话 ID。

## 变更影响分析

- **会话标识兼容性提升**：某些平台或自定义 Adapter 可能产生包含 `:` 的 `session_id`。配置路由现在可以处理这类 UMO，减少因会话 ID 字符集差异导致的路由失效。
- **插件拆分 UMO 的最佳实践更明确**：AI 开发者在生成插件代码时，不应使用 `umo.split(":")` 后假设长度恒为 3；应使用 `umo.split(":", 2)` 或优先依赖 `event.session_id`、`event.get_platform_id()` 等事件属性。
- **通配符匹配边界更清晰**：通配符只作用于 `platform_id`、`message_type`、完整 `session_id` 三个逻辑字段。即使 `session_id` 内部含有冒号，也不会被进一步拆成多个可独立匹配的字段。
- **错误判断更稳定**：`update_routing_data`、`update_route`、`delete_route` 的格式校验与 `_is_umo_match` 使用一致的解析规则，避免出现“更新时合法、匹配时非法”或相反的不一致状态。
- **不新增对外 API**：此次变更没有引入新的插件装饰器、Message Chain 组件、Provider 接口或 Adapter 接口。它是对现有 UMO 字符串处理契约的兼容性增强。
