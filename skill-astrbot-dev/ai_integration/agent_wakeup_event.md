---
title: Agent 唤醒回复投递与流式缓冲 (Agent Wakeup Delivery)
type: improvement
status: stable
last_updated: 2026-11-01
related_base: agent/agent-runner.md
---

## 概述

Agent 唤醒（后台任务完成后的主 Agent 唤醒）不再静默消费 Agent 输出。`AgentWakeupEvent` 的最终回复现在通过正常消息管道投递，并且唤醒事件显式关闭流式输出，将流式片段缓冲为一条完整的 `MessageChain` 后发送。

同时，`run_agent(...)` 移除了 `emit_output` 静默开关。此前该开关用于无头模式下抑制 `agent_stats`、`tool_call`、`reasoning`、`streaming_delta`、`tool_direct_result` 等自动 UI 帧，并让异常直接向上抛出。变更后，`run_agent(...)` 统一把 Agent 输出转发给消息管道，唤醒路径不再依赖 `emit_output=False` 实现静默执行。

## 核心契约

### 1. `run_agent(...)` 输出语义

`astrbot/core/astr_agent_run_util.py` 中的 `run_agent(...)` 不再接受 `emit_output` 参数。其行为统一为：

- 不再根据 `emit_output` 短路 `agent_stats` 的 webchat 发送。
- 不再根据 `emit_output` 短路 `tool_call`、`tool_direct_result`、`reasoning`、`streaming_delta` 与最终 `llm_result` 的输出链。
- `can_buffer_llm_result` 直接由 `_should_buffer_llm_result(buffer_intermediate_messages, stream_to_general, agent_runner, ...)` 决定，不再与 `emit_output` 做逻辑与。
- 异常处理不再保留 `if not emit_output: raise` 的无头特殊分支，统一走消息管道路径。

因此，调用方不能再用 `run_agent(..., emit_output=False)` 表示“静默执行并丢弃自动输出”。需要投递的 Agent 输出会进入正常管道；需要隐藏输出的场景应由事件层或平台适配层处理，而不是由 `run_agent` 的静默开关处理。

### 2. `AgentWakeupEvent` 流式缓冲

`astrbot/core/platform/agent_wakeup_event.py` 中，`AgentWakeupEvent.__init__(...)` 新增：

```python
self.set_extra("enable_streaming", False)
```

这表示唤醒事件不支持流式输出。`AgentWakeupEvent.send_streaming(...)` 的行为改为：

1. 遍历上游 `generator`。
2. 跳过空 `MessageChain`，即 `chain.chain` 为空的链。
3. 第一个非空链作为缓冲区；后续非空链通过 `buffer.chain.extend(chain.chain)` 追加。
4. 遍历结束后，如果缓冲区存在，调用 `buffer.squash_plain()` 合并纯文本段。
5. 调用 `await self.send(buffer)` 一次性投递完整消息。

`use_fallback` 仅作为消息事件接口的兼容参数，唤醒事件始终缓冲流。若整个流为空，或只包含 `MessageChain(type="break")` 这类空控制帧，则不会调用 `send(...)`，也不会记录一次成功发送。

### 3. 唤醒运行路径

`astrbot/core/pipeline/process_stage/method/agent_sub_stages/internal.py` 移除了原先的唤醒专用分支：

```python
aasync for _ in run_agent(agent_runner, self.max_step, emit_output=False):
    pass
```

该分支此前会驱动 Agent 但丢弃所有输出。变更后，唤醒事件与其他事件一样执行正常 Agent 运行路径，最终回复可以进入消息管道。

唤醒结束后的状态检查被移动到 `final_resp = agent_runner.get_final_llm_resp()` 之后：

- 若 `event.is_stopped()` 或后台停止信号已置位，抛出 `asyncio.CancelledError()`。
- 若 `agent_runner.state == AgentState.ERROR` 或最终响应 `role == "err"`，抛出 `RuntimeError`，并在错误信息中附加最终响应文本。

这保持了唤醒运行的停止与错误可观测性，同时不再把最终回复作为“仅内部可见”的内容丢弃。

### 4. 后台任务提示词语义

`astrbot/core/astr_main_agent_resources.py` 中后台任务相关提示词从：

```text
Use `send_message_to_user` to deliver any message; your final response alone is not delivered.
```

改为：

```text
You can use `send_message_to_user` to send images, files, and other generated artifacts.
```

这意味着后台任务的最终回复现在会被正常投递。`send_message_to_user` 的定位变为发送图片、文件等生成物或需要主动推送的附加内容，而不是发送任何消息的唯一通道。

## 数据流

新的唤醒回复数据流为：

1. 后台任务完成后构造并投递唤醒事件到事件队列。
2. 事件泵消费 `AgentWakeupEvent`，进入正常 Agent 运行路径。
3. Agent Runner 执行工具循环并产生 `AgentResponse`。
4. `run_agent(...)` 将输出链转发给消息管道，不再有 `emit_output` 静默短路。
5. `AgentWakeupEvent` 设置 `enable_streaming=False`，因此流式内容通过 `send_streaming(...)` 缓冲。
6. 非空流式片段被合并为一条 `MessageChain`，经 `squash_plain()` 后通过 `AgentWakeupEvent.send(...)` 投递。
7. 最终回复与工具或钩子内部显式调用 `send_message_to_user` 产生的内容都可以触达用户。

该设计使唤醒事件与普通消息事件共用同一条主 Agent 构建、运行与投递路径，降低后台任务与交互式消息在输出语义上的分叉。

## 与统一消息模型的关系

本次变更没有新增 Message Chain 组件，也没有新增 Provider 方法、装饰器或 Platform Adapter 注册接口。`MessageChain` 仍是唯一消息结构，`AgentWakeupEvent.send(...)` 仍是唤醒事件投递入口。

`MessageChain(type="break")` 是 Agent 流式输出中的控制帧。对于普通流式平台，适配器可能解释该控制帧并关闭当前流式卡片；对于唤醒事件，`send_streaming(...)` 会直接跳过空链，因此 `break` 不会产生可见消息，也不会单独触发发送记账。

## 变更影响分析

- **后台任务最终回复变为用户可见**：唤醒路径不再静默丢弃 Agent 输出。AI 开发者在分析后台任务时，应预期最终回复会通过正常管道发送，而不是必须依赖 `send_message_to_user` 才能让用户看到结果。
- **`emit_output` 不再是可用契约**：旧文档或旧代码中的 `run_agent(..., emit_output=False)` 已失效。生成代码或排查日志时，不应再假设存在无头静默开关；唤醒静默语义已由事件类型与流式缓冲策略替代。
- **唤醒事件显式关闭流式输出**：`AgentWakeupEvent` 设置 `enable_streaming=False`，且 `send_streaming(...)` 会缓冲后一次性发送。依赖逐 token 流式展示的 AI 助手不应把唤醒路径理解为流式交互路径。
- **空流与 break 帧不会触发发送**：只有空 `MessageChain` 或 `MessageChain(type="break")` 的流不会产生消息，也不会记录成功发送。排查“唤醒没有消息”时，应区分“没有产生非空输出”与“发送失败”。
- **唤醒错误会进入投递链路**：`run_agent(...)` 不再在无头模式下直接向上抛出异常，唤醒错误文本可以进入正常回复投递；同时 `internal.py` 仍会在最终响应为错误状态时抛出 `RuntimeError` 用于追踪。失败不再等同于静默。
- **显式发送与最终回复可能同时出现**：工具或钩子内部调用 `send_message_to_user` 仍会立即投递，随后唤醒的最终回复也会投递。插件如果同时使用显式发送和最终总结，应预期用户可能看到两条或更多消息，并据此设计提示词与输出格式。
- **后台提示词语义变化**：后台任务提示词不再声明“最终回复不会被投递”，而是把 `send_message_to_user` 定位为发送图片、文件等生成物。生成后台任务提示词时，不应继续写入“必须调用 send_message_to_user 才能交付最终文本”的旧指令。
- **不引入新的插件 API**：本次变更没有新增装饰器、Message Chain 组件、Provider 接口或 Adapter 注册接口。它改变的是 `run_agent(...)` 的输出门控语义、`AgentWakeupEvent` 的发送策略，以及唤醒事件在主 Agent 执行路径中的投递方式。