---
title: 分段回复对数基数鲁棒性（Segmented Reply Log Base）
type: improvement
status: stable
last_updated: 2026-09-27
related_base: design_standards/event_flow.md
---

## 概述

AstrBot Pipeline 的 Respond Stage 在计算分段回复间隔时，会对配置项 `platform_settings.segmented_reply.log_base` 进行合法性校验。此前该值仅通过 `float(...)` 直接被用于 `_calc_comp_interval` 中的对数计算；若用户将 `log_base` 配置为 `1`、`0`、负数或非数值，`math.log` 会抛出 `ZeroDivisionError` / `ValueError`，导致整个回复在第一次发送前被丢弃。

本次变更引入 `_resolve_log_base(value)` 辅助函数，将不可用的 `log_base` 收敛为一个文档化的默认值 `DEFAULT_LOG_BASE = 2.6`，从而让分段回复在配置异常时仍可正常发送。

该变更位于 Pipeline Respond Stage 的初始化与运行时决策层，不改变 Message Chain、Provider、Adapter 或事件分发接口，也不新增任何对外 API。

## 核心行为

实现位置：

```python
astrbot/core/pipeline/respond/stage.py
```

新增模块级常量与函数：

```python
DEFAULT_LOG_BASE = 2.6

def _resolve_log_base(value: float | str | None) -> float:
    ...
```

`_resolve_log_base` 的稳定行为为：

1. 尝试 `float(value)` 转换。若转换失败（`TypeError` / `ValueError`），记录 error 日志并返回 `DEFAULT_LOG_BASE`。
2. 转换成功后，检查结果是否满足 `math.isfinite(log_base)` 且 `log_base > 1`。
3. 若不满足（例如 `1`、`0`、负数、`NaN`、`inf`），记录 error 日志并返回 `DEFAULT_LOG_BASE`。
4. 否则返回原值。

Respond Stage 的初始化逻辑由直接 `float(...)` 改为：

```python
self.log_base = _resolve_log_base(
    ctx.astrbot_config["platform_settings"]["segmented_reply"]["log_base"],
)
```

## 为什么需要保护 `log_base`

`math.log(words, base)` 对 base 有严格定义域要求：

- `base == 1` 会触发 `ZeroDivisionError`（因 `log(1) == 0` 出现在分母）。
- `base <= 0` 或 `base` 非有限值会触发 `ValueError`。

旧实现中，这两个异常从 `_calc_comp_interval` 直接冒泡到 Respond Stage 的分段发送流程，且在第一次 `send` 之前抛出，导致本轮 AI 回复完全不会发送给用户。新的 `_resolve_log_base` 把这类“配置错导致整条回复丢失”的跨层耦合在 Respond Stage 边界消除：无效值不会中断发送路径，而是显式回退到 `2.6`。

## 配置契约

`platform_settings.segmented_reply.log_base` 仍是分段回复间隔计算的对数基数。稳定语义为：

- 缺失、非法类型、非有限值或 `<= 1`：被夹取为默认值 `2.6`。
- 有限且 `> 1` 的数值：原值生效。

该字段仍属于 `platform_settings.segmented_reply` 命名空间，与 `interval_method` 一起决定分段回复的节奏。Respond Stage 只负责解释该字段为可用的对数基数，不改变上层对分段回复的语义约定。

## 与消息流转模型的关系

此变更位于 `design_standards/event_flow.md` 描述流程的第 7 步“结果装饰”之后的实际发送阶段。Respond Stage 消费 `platform_settings.segmented_reply` 决定何时把一次逻辑回复拆成多次物理发送。

数据流为：

1. 插件或主 Agent 产生一次性回复。
2. Respond Stage 初始化时通过 `_resolve_log_base(...)` 得到可用的 `self.log_base`。
3. 分段发送过程中调用 `_calc_comp_interval` 计算段间间隔。
4. 分段回复按照间隔逐条调用 `event.send(...)` 等发送路径。
5. 若 `segmented_reply` 相关配置无效，仍能完成整个回复发送，只是间隔节奏回退到默认对数基数。

该设计保持了 Message Chain、`AstrMessageEvent` 与 Adapter 发送接口的职责边界：分段节奏的健壮性在 Stage 层闭环，插件与 Adapter 无需感知。

## 变更影响分析

- **配置错误不再导致整条回复丢失**：当用户将 `segmented_reply.log_base` 清空、填成 `1`、`0`、负数或非数值时，Respond Stage 会回退到 `DEFAULT_LOG_BASE = 2.6`，而不是让 `_calc_comp_interval` 的异常在首次 `send` 前终止整个回复流程。AI 助手在排查“AI 回复完全不出现”类问题时，应将 `log_base` 作为候选检查项，但要注意其异常路径现在已被降级而非崩溃。
- **默认语义变为 `2.6`**：配置值 `0`、负数、`1`、`NaN`、`inf` 以及无法 `float()` 的字符串都会收敛到 `2.6`。生成配置校验或文档时不应假设“`1` 表示关掉对数缩放”之类的语义仍然存在，`<= 1` 一律视为不可用。
- **error 日志是显式降级信号**：任何回退都会通过 `logger.error` 记录原始 `value` 与 `DEFAULT_LOG_BASE`。AI 助手在分析运行日志时，应把该 error 视为“配置无效但系统已自愈”，而不是致命错误。
- **Respond Stage 初始化不再抛出 `ZeroDivisionError` / `ValueError`**：该 Stage 的 `initialize` 阶段现在把 `float(...)` 之外的定义域检查也纳入统一入口，因此不会因为 `log_base` 异常阻断 Pipeline 初始化或 Stage 注册。
- **不新增对外 API**：本变更没有新增装饰器、Message Chain 组件、Provider 方法、Adapter 接口或配置 schema 字段。`_resolve_log_base` 与 `DEFAULT_LOG_BASE` 是 `astrbot/core/pipeline/respond/stage.py` 的内部实现细节，不构成跨模块契约。
- **与其他 Stage 配置校验保持独立**：`_resolve_log_base` 针对的是 Segmentation 的对数基数这一特定定义域，不替其他 Stage 或 `interval_method` 做通用数值校验。生成其他配置读取逻辑时，不应假设 AstrBot 已提供统一的正数下界工具。
- **分段回复间隔仍是 `log_base` 的函数**：回退后分段节奏与用户预期可能不同。AI 助手在分析“分段回复太慢/太快”时，需要先确认 `log_base` 是否被回退到 `2.6`，例如 `ValueError` / `TypeError` 触发的 error 日志。
- **不改变事件生命周期**：`on_decorating_result` 钩子、`after_message_sent` 钩子、`event.stop_event()` 等事件控制流程未受影响。分段回复的健壮性提升仅发生在 Respond Stage 计算间隔的局部路径。
