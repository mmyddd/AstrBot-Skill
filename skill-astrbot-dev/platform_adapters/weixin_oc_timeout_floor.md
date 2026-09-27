---
title: 微信个人号适配器超时下界与配置鲁棒性
type: improvement
status: stable
last_updated: 2026-09-27
related_base: platform_adapters/adapter_interface.md
---

## 概述

微信个人号 Platform Adapter (`WeixinOCAdapter`) 在读取用户可调的数值型超时配置时，引入了统一的整数解析与最小值（下界）约束。该变更防止 Dashboard 清空数值输入框（写入 `0`）或用户填写负数/非数值时，把适配器的 HTTP 超时意外地转换为“无超时”，从而导致接收循环无限阻塞。

该变更不新增 Message Chain 组件，不改变插件侧事件接收与发送的入口（`AstrMessageEvent`、`event.send(...)`、`event.plain_result(...)` 均不受影响），只调整 `WeixinOCAdapter` 在构造阶段对平台配置的解释方式。受影响的两个用户可调字段是：

- `weixin_oc_qr_poll_interval`
- `weixin_oc_long_poll_timeout_ms`
- `weixin_oc_api_timeout_ms`

## 核心行为

适配器新增类级常量，用于表达“HTTP 超时必须有下界”的契约：

```python
class WeixinOCAdapter(Platform):
    MIN_TIMEOUT_MS = 1_000
```

构造阶段不再直接调用 `int()` 或 `max()` 处理原始配置，而是统一通过内部辅助方法读取并夹取值：

```python
self.qr_poll_interval = self._get_int_config(
    "weixin_oc_qr_poll_interval",
    1,
    1,
)
self.long_poll_timeout_ms = self._get_int_config(
    "weixin_oc_long_poll_timeout_ms",
    35_000,
    self.MIN_TIMEOUT_MS,
)
self.api_timeout_ms = self._get_int_config(
    "weixin_oc_api_timeout_ms",
    120_000,
    self.MIN_TIMEOUT_MS,
)
```

可概括的稳定行为为：

- 字段缺失：使用默认值（`1` / `35_000` / `120_000`）。
- 字段值为非数值（例如 `""`、`None`）：回退到默认值，而不是抛出异常。
- 字段值为数值但低于下界（例如 `0`、负数、`200` 毫秒）：被夹取到下界。
- `weixin_oc_qr_poll_interval` 下界为 `1`；两个超时字段下界为 `MIN_TIMEOUT_MS`（`1000` 毫秒）。
- 合法且高于下界的值保持原样（例如 `5_000`、`45_000`）。

`api_timeout_ms` 最终会以 `aiohttp.ClientTimeout(total=...)` 的形式传给底层客户端，测试通过 `adapter.client.api_timeout_ms` 断言真实下发给 aiohttp 的值也满足下界。

## 为什么下界是必要的

Aiohttp 将 `ClientTimeout(total=0)` 解释为“无超时”。Dashboard 的通用数值组件在用户清空输入框时，会把空字符串通过 `parseFloat('')` 映射为 `0` 后写入配置。因此，一个被清空的超时字段会让接收循环中的 `getupdates` 长连接永久等待，而适配器本身是靠 `except asyncio.TimeoutError` 重试分支来维持长轮询健壮性的——一旦 `total=0`，该分支永远不会被触发。

将用户可调超时强制约束到正的毫秒值，使“Dashboard 写入 `0`”这一前端行为不再等价于“关闭 HTTP 超时”，从而在应用层消除该跨层耦合。

## 与统一消息模型的关系

此变更完全位于 Platform Adapter 的构造阶段，不涉及 `AstrBotMessage` 的字段填充、`convert_message` 逻辑、`MessageSession` 构造、`commit_event` 提交，也不改变 `AstrMessageEvent` 的发送接口。

插件作者看到的仍是标准契约：

```python
@filter.command("echo")
async def echo(self, event: AstrMessageEvent):
    yield event.plain_result(event.message_str)
```

微信个人号适配器的超时与长轮询参数由适配器自身消费，不会作为新的 Message Chain 组件、Provider 参数或事件属性对外暴露。

## 变更影响分析

- **配置清空不再禁用 HTTP 超时**：`weixin_oc_api_timeout_ms` 与 `weixin_oc_long_poll_timeout_ms` 被清空（写入 `0`）时，适配器会夹取到 `1000` 毫秒下界，而不是向 aiohttp 传递 `ClientTimeout(total=0)`。AI 开发者在排查微信个人号“请求挂死”类问题时，应先确认适配器实际持有的 `api_timeout_ms` / `long_poll_timeout_ms`，而不是只看 Dashboard 中的原始值。
- **负数与低于下界的值被夹取而非拒绝**：负数（如 `-1`、`-35_000`）以及低于下界的正数（如 `200`）都会被提升到 `MIN_TIMEOUT_MS`。生成配置或文档时不应假设“非零即生效”，而应理解下界为 `1000` 毫秒。
- **非数值字段不再使平台初始化失败**：`weixin_oc_api_timeout_ms=""`、`weixin_oc_long_poll_timeout_ms=None`、`weixin_oc_qr_poll_interval=""` 都会回退到默认值而不是让 `PlatformManager.initialize` 因 `invalid literal` 之类错误中止加载。AI 助手在生成平台配置校验逻辑时，应保留“非数值回退到默认值”的语义，而不是直接抛出异常。
- **默认值语义保持稳定**：未设置字段时，`qr_poll_interval` 回退到 `1`，`long_poll_timeout_ms` 回退到 `35_000`，`api_timeout_ms` 回退到 `120_000`。AI 开发者在分析微信个人号默认轮询节奏时不应把默认长轮询延迟理解为可变字段。
- **对 aiohttp 的实际下发值有保证**：`adapter.client.api_timeout_ms` 会反映下界后的实际值。排查超时行为时，应优先读取该属性而非原始 `platform_config` 字段。
- **插件 API 与适配器注册接口没有变化**：本变更没有新增装饰器、Message Chain 组件、Provider 方法、`register_platform_adapter` 参数或 `AstrMessageEvent` 字段。它只改变 `WeixinOCAdapter.__init__` 对配置的解释契约。
- **不影响适配器作者的自定义 Platform**：`MIN_TIMEOUT_MS` 与 `_get_int_config(...)` 是 `WeixinOCAdapter` 的内部约定，未提升为跨适配器的通用接口。AI 助手在生成其他 Platform Adapter 的超时读取逻辑时，应参考其“正下界 + 非数值回退”的思路，但不应假设基类已提供同名辅助方法。
