---
title: Agent Stop 中断工具等待行为
type: improvement
status: stable
last_updated: 2026-06-24
related_base: agent/agent-runner.md
---

## 概述

`ToolLoopAgentRunner` 的停止请求现在统一通过异步中断信号处理，并会在工具执行等待阶段生效。该变更的目标是：当 Agent 或 Subagent 正在等待 Tool 执行结果时，如果运行过程收到 stop request，Runner 不再只等待当前工具自然返回，而是可以进入中断收尾流程。

这影响 `context.tool_loop_agent(...)` 背后的 Agent Runner 行为，尤其是使用 FunctionTool、Subagent handoff、长耗时工具或异步工具执行器的场景。插件侧调用 `tool_loop_agent(...)` 的签名没有变化，但停止请求下的运行时语义更明确：停止会传播到 LLM 请求和工具执行链路，并最终生成 aborted 类型的 AgentResponse。

## 核心行为

`ToolLoopAgentRunner` 在 reset 阶段初始化内部中断信号：

```python
self._abort_signal = asyncio.Event()
```

停止请求入口改为设置该信号：

```python
def request_stop(self) -> None:
    self._abort_signal.set()
```

Runner 内部通过统一判断方法读取停止状态：

```python
def _is_stop_requested(self) -> bool:
    return self._abort_signal.is_set()
```

当停止发生在 LLM 响应阶段或工具执行阶段时，Runner 会进入统一的中断收尾方法：

```python
async def _finalize_aborted_step(
    self,
    llm_resp: LLMResponse | None = None,
) -> AgentResponse:
    ...
```

该方法会设置：

- `self.final_llm_resp`
- `self._aborted = True`
- Runner 状态为 `AgentState.DONE`
- `self.stats.end_time`

并返回：

```python
AgentResponse(
    type="aborted",
    data=AgentResponseData(chain=MessageChain(type="aborted")),
)
```

如果中断时需要向上下文写入 assistant 消息，统一使用系统中断提示：

```python
USER_INTERRUPTION_MESSAGE = (
    "[SYSTEM: User actively interrupted the response generation. "
    "Partial output before interruption is preserved.]"
)
```

## 中断信号的数据流

本次变更把同一个 `abort_signal` 传入两个关键链路。

### 1. LLM 请求迭代链路

Runner 构造 LLM 请求参数时新增：

```python
"abort_signal": self._abort_signal
```

这意味着 Provider 调用路径可以感知同一个停止信号。该变更不新增插件侧 Provider API，但让 Agent Runner 与 Provider 请求之间共享可取消状态。

### 2. Tool 执行链路

工具执行请求中同样传入：

```python
abort_signal=self._abort_signal
```

同时，工具执行结果的异步迭代从直接遍历 executor：

```python
async for resp in executor:
```

变为通过 Runner 的封装迭代：

```python
async for resp in self._iter_tool_executor_results(executor):
```

当工具执行等待被停止请求打断时，Runner 使用内部异常表示该状态：

```python
class _ToolExecutionInterrupted(Exception):
    """Raised when a running tool call is interrupted by a stop request."""
```

`_ToolExecutionInterrupted` 会向上保留传播，而不是被普通工具异常处理逻辑吞掉。外层工具处理流程捕获该异常后，会调用 `_finalize_aborted_step(...)` 生成 aborted 响应并结束当前 Agent step。

## 与 Subagent / Handoff 的关系

提交信息明确该修复面向 Subagent 工具等待中断：

```text
fix(core): interrupt subagent tool waits on stop (#5850)
```

在 AstrBot 的 Agent 模型中，Subagent 通常通过 handoff tool 或 agent-as-tool 方式被主 Agent 调用。主 Agent 等待 Subagent 工具返回时，本质上仍处于工具执行等待阶段。本次变更使 stop request 能进入该等待链路，避免主 Agent 因正在等待 Subagent tool 而无法及时收尾。

插件作者仍应按照现有 Agent / Tool 契约定义工具：

- 使用 `FunctionTool` 定义 Tool。
- 使用 `context.tool_loop_agent(...)` 调用工具循环 Agent。
- 对长耗时工具保持异步实现，避免阻塞 Event Loop。

该变更不要求插件手动读取 `_abort_signal`，也没有把 `_ToolExecutionInterrupted` 作为插件可捕获的公开异常暴露。它是 Runner 内部对停止请求传播能力的增强。

## 对插件开发逻辑的影响

对于插件作者，最重要的稳定行为是：如果用户或上层流程请求停止正在运行的 Agent，`tool_loop_agent(...)` 背后的 Runner 现在可以在工具等待阶段进入 aborted 收尾，而不是必须等工具执行链路自然完成。

这对以下插件场景尤其重要：

- Tool 内部调用外部服务，响应时间较长。
- Tool 触发 Subagent handoff，主 Agent 正在等待子智能体返回。
- Tool executor 通过异步迭代逐步产生结果。
- 用户在 Agent 仍在调用工具时主动中断生成。

插件不应依赖“停止请求后工具一定继续跑到自然结束”的旧行为。若插件工具包含必须完成的清理逻辑，应放在工具自身的 `try/finally` 或插件 `terminate()` 生命周期中，而不是假设 Agent Runner 会等待所有工具逻辑完整结束。

## 变更影响分析

- **停止状态来源统一**：Runner 不再维护独立的 `_stop_requested` 布尔值，而是通过 `asyncio.Event` 形式的 `_abort_signal` 表达停止状态。AI 开发者在理解 Agent Runner 时，应把 stop request 视为可传播的异步信号，而不是单点布尔标记。
- **工具等待可被中断**：长耗时 Tool、Subagent handoff 或异步 Tool executor 在等待结果时可能被中断并进入 aborted 收尾。生成插件代码时，不应假设工具调用一旦开始就一定会完整返回后再结束 Agent step。
- **中断不是普通工具错误**：`_ToolExecutionInterrupted` 会被保留传播，并触发 Runner 的 aborted 收尾；它不应被理解为 Tool 业务失败，也不等同于工具返回 `is_error=True`。
- **中断响应保留部分输出语义**：当 LLM 已有部分输出时，Runner 使用 `USER_INTERRUPTION_MESSAGE` 表示用户主动中断，且提示中明确 partial output before interruption is preserved。AI 助手在分析历史消息时，应把该文本视为系统中断标记，而不是普通助手业务回复。
- **Agent Hooks 的完成阶段仍可能被触发**：中断收尾方法会设置最终 LLMResponse 并进入 DONE 状态；已有 Agent 生命周期理解应覆盖 aborted 结束路径，而不只覆盖正常完成路径。
- **插件 API 签名不变**：本变更没有新增 `context.tool_loop_agent(...)` 参数，也没有新增 Message Chain 组件、Provider 类型或 Adapter 接口。它改变的是 Agent Runner 内部在 stop request 下对 LLM 请求和工具执行链路的协调方式。
- **长耗时工具仍应避免阻塞 Event Loop**：如果 Tool 使用同步阻塞 I/O，即使 Runner 有 `_abort_signal`，阻塞代码也可能无法及时让出控制权。插件作者仍应使用 async I/O 或线程池封装阻塞操作。
- **Subagent handoff 更可控**：主 Agent 等待子智能体时的停止响应能力增强。AI 开发者在设计多智能体插件时，可以把 stop request 理解为会穿过主 Agent 与 handoff tool 等待边界的运行时控制信号。 
