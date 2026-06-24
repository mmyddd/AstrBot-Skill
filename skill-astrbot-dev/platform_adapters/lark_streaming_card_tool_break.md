---
title: Lark 流式卡片延迟创建与工具调用分段
type: improvement
status: stable
last_updated: 2026-06-24
related_base: platform_adapters/adapter_interface.md
---

## 概述

Lark Platform Adapter 的流式发送逻辑现在会延迟创建 CardKit 流式卡片，并在工具调用分段信号到达时关闭当前卡片，后续文本 token 到来后再创建新的流式卡片。

该变更影响 `AstrMessageEvent.send_streaming(...)` 在 Lark 平台上的运行时表现，尤其是 Agent 流式输出过程中同时展示工具调用状态消息的场景。插件作者仍然使用标准 Message Chain、`event.send(...)`、`event.send_streaming(...)` 或 Agent 流式输出能力，不需要直接调用 Lark CardKit API。

核心目标是避免两个问题：

1. LLM 尚未产生文本 token、仅进入工具调用阶段时，Lark 不再提前渲染空白流式卡片。
2. 当工具状态消息需要独立发送时，Lark 会结束当前流式卡片，使工具状态消息与后续模型文本在消息顺序上保持正确。

## Agent 分段信号的触发条件

Agent 运行工具调用响应时，`run_agent` 仍使用 `MessageChain(type="break")` 表达流式分段边界：

```python
yield MessageChain(chain=[], type="break")
```

但该分段信号现在只在以下条件同时满足时发送：

```python
if agent_runner.streaming and show_tool_use:
    yield MessageChain(chain=[], type="break")
```

稳定行为为：

- `agent_runner.streaming == True` 且 `show_tool_use == True`：发送空 Message Chain，`type="break"`，用于通知下游平台关闭当前流式消息段。
- `show_tool_use == False`：不发送 `break`，因为不会插入独立的工具状态消息，AI 回复可以继续保留在同一个流式消息中。

这里的 `break` 不是新的消息组件，也不携带业务内容。它是流式发送链路中的控制型 Message Chain，用于表达“当前流式输出段应在此处断开”。

## Lark send_streaming 行为

Lark 事件的核心方法仍是：

```python
async def send_streaming(self, generator, use_fallback: bool = False)
```

新的稳定流程为：

1. 进入 `send_streaming(...)` 时不立即创建 CardKit 流式卡片。
2. 适配器开始消费上游 `generator` 产出的 Message Chain。
3. 当收到第一个 `Plain` 文本组件后，才调用 `_create_streaming_card()` 创建流式卡片。
4. 创建成功后调用 `_send_card_message(...)` 发送卡片消息。
5. 后续文本通过 `_update_streaming_text(card_id, delta, sequence)` 更新到当前卡片。
6. 流式输出结束时，补发最终文本并调用 `_close_streaming_mode(card_id, sequence)` 关闭流式模式。

因此，如果一次流式响应完全没有产生文本内容，Lark Adapter 不会创建空卡片。它只执行发送收尾所需的副作用，例如 Metric 上报和 `_has_send_oper` 标记。

## break 分段处理

当 Lark Adapter 在流式 generator 中收到：

```python
chain.type == "break"
```

且当前已经存在流式卡片与发送任务时，会执行以下行为：

1. 标记当前发送循环结束。
2. 唤醒并等待 sender loop 完成。
3. 通过 `_flush_and_close_card()` 补发最终文本并关闭当前卡片流式模式。
4. 重置 `card_id`、`sequence`、`delta`、`last_sent`、`done`、`sender_task` 等状态。
5. 继续消费 generator。
6. 后续第一个新的 `Plain` 文本 token 到来时，再懒创建新的 CardKit 流式卡片。

这使 Lark 的消息顺序更符合 Agent 数据流：

```text
模型前半段流式卡片
工具状态消息
模型后半段流式卡片
```

如果 `show_tool_use=False`，上游不会发送 `break`，Lark 也不会因为工具调用边界拆分卡片。此时模型输出会继续留在同一张流式卡片中。

## 回退路径

如果第一个文本 token 到来后，Lark Adapter 创建或发送流式卡片失败，会进入非流式回退路径。

回退逻辑会：

1. 使用已收到的初始文本构造 `MessageChain().message(initial_text)`。
2. 继续消费 generator 中剩余的 Message Chain。
3. 将后续组件追加到同一个缓冲 Message Chain 中。
4. 调用 `buffer.squash_plain()` 合并纯文本段。
5. 使用 `await self.send(buffer)` 进行普通发送。
6. 执行 Metric 上报并设置 `_has_send_oper=True`。

该回退只发生在流式卡片创建或发送失败时，不改变插件侧构造 Message Chain 的方式。

## 与统一消息模型的关系

此变更没有新增 Message Component。插件侧仍然应使用 `Plain`、`Image`、`MessageChain` 等统一消息模型表达回复内容。

`MessageChain(type="break", chain=[])` 是 Agent 流式输出到 Platform Adapter 之间的控制信号，不是插件作者应主动构造的业务消息。Lark Adapter 负责解释该控制信号，并把它映射为 CardKit 流式卡片的关闭与重新创建。

该设计保持了 AstrBot 的分层边界：

- Agent 层决定是否需要插入工具状态消息，并在需要时发送 `break` 控制信号。
- Platform Adapter 层决定如何把 `break` 映射到平台原生的流式消息能力。
- 插件层继续消费和生成统一的 Message Chain，不需要依赖 Lark 专属 SDK。

## 变更影响分析

- **Lark 不再提前创建空流式卡片**：流式卡片创建延迟到第一个 `Plain` 文本 token 到达后执行。AI 开发者在分析 Lark 流式输出时，应理解工具调用阶段可能暂时没有卡片出现，这是预期行为。
- **工具状态显示会影响分段**：只有 `show_tool_use=True` 且 Agent 处于 streaming 模式时，上游才会发送 `MessageChain(type="break")`。这表示工具状态消息需要作为独立消息插入，Lark Adapter 会关闭当前卡片并等待后续文本创建新卡片。
- **隐藏工具状态时不会拆分卡片**：当 `show_tool_use=False` 时，即使 Agent 内部发生工具调用，也不会发送 `break` 控制信号。Lark 上的 AI 回复会尽量保持在同一个流式卡片中。
- **`break` 不是可发送内容**：`type="break"` 的 Message Chain 是空链控制信号，不是新的 Message Component。插件作者不应把它用于表达文本、图片或其他业务消息。
- **Lark 平台现在支持工具调用边界的流式顺序控制**：收到 `break` 后，当前 CardKit 流式卡片会被关闭，后续文本会在新的卡片中展示，从而让工具状态消息位于两个模型输出段之间。
- **无文本输出不会产生空卡片**：如果 generator 没有产出任何 `Plain` 文本，Lark Adapter 不会创建 CardKit 卡片，只执行必要的发送收尾副作用。这降低了空回复或纯控制流导致的视觉噪音。
- **回退路径仍会消费剩余输出**：当卡片创建或发送失败时，Lark Adapter 会继续消费 generator 并合并为普通 Message Chain 发送。AI 开发者排查 Lark 流式失败时，应预期最终可能退化为一次非流式消息。
- **插件 API 没有变化**：此变更没有新增装饰器、Provider 方法、Message Component 或 Adapter 注册接口。它改变的是 Agent 流式控制信号与 Lark Platform Adapter CardKit 发送实现之间的协作方式。