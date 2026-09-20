---
title: Local Runtime 网络策略提示注入 (Local Network Policy Notice Injection)
type: improvement
status: stable
last_updated: 2026-09-20
related_base: agent/agent-runner.md
---

## 概述

在 `computer_use_runtime == "local"` 的 Agent 运行时，当 Local 执行权限允许执行代码/命令但不允许网络访问时，AstrBot 现在会把网络策略提示写入 **系统提示词 (system_prompt)**，而不是把该提示附加在工具调用返回值中。

核心提示常量：

```python
from astrbot.core.tools.computer_tools.util import LOCAL_NETWORK_POLICY_NOTICE
```

该变更同时影响两条 Agent 构建路径：

- `build_main_agent(...)`：主 Agent 构建阶段。
- `Context.tool_loop_agent(...)`：插件或核心代码显式调用工具循环 Agent 的阶段。

变更的本质是**策略提示的注入位置**从“工具结果前缀”迁移到“系统提示词”，从而把“本地运行时受网络限制”这一事实提前告知 LLM，而不是等工具返回后再补充说明。

## 权限判定依赖

注入决策依赖 Local 权限策略对象：

```python
from astrbot.core.tools.computer_tools.util import (
    LOCAL_NETWORK_POLICY_NOTICE,
    get_local_permission_policy,
    is_local_runtime,
)

local_policy = get_local_permission_policy(run_context)
```

`local_policy` 暴露的关键字段：

- `local_policy.allow_execution`：当前会话角色是否允许在 Local runtime 执行代码/命令。
- `local_policy.allow_network`：当前会话角色是否允许访问网络。
- `local_policy.requires_sandbox`：是否需要沙盒（shell 工具仍在使用）。
- `local_policy.filesystem_scope`：文件系统可访问范围（sandbox roots 相关）。

注入条件为：

```python
if local_policy.allow_execution and not local_policy.allow_network:
    # 注入 LOCAL_NETWORK_POLICY_NOTICE
```

即：**允许执行但禁止网络** 时才注入网络策略提示。`allow_network=True` 或 `allow_execution=False` 时都不注入。

## build_main_agent 路径

在 `build_main_agent(...)` 中，当 `config.computer_use_runtime == "local"` 时，构建完 `astr_agent_ctx` 后立即查询权限策略：

```python
if config.computer_use_runtime == "local":
    local_policy = get_local_permission_policy(
        AgentContextWrapper(context=astr_agent_ctx)
    )
    if (
        local_policy.allow_execution
        and not local_policy.allow_network
        and LOCAL_NETWORK_POLICY_NOTICE not in (req.system_prompt or "")
    ):
        req.system_prompt = (
            f"{req.system_prompt or ''}\n{LOCAL_NETWORK_POLICY_NOTICE}\n"
        )
```

稳定行为：

1. 只有在 Local runtime 下才进行提示注入。
2. 只在 `allow_execution=True` 且 `allow_network=False` 时注入。
3. 注入是**幂等**的：如果 `req.system_prompt` 中已包含 `LOCAL_NETWORK_POLICY_NOTICE`，不会重复添加。
4. 注入位置是把提示追加到现有 `system_prompt` 末尾（前后各有一个换行）。

## Context.tool_loop_agent 路径

插件或核心通过 `Context.tool_loop_agent(...)` 直接发起工具循环 Agent 时，同样在 Agent 运行前评估策略。

注入前置条件更严格：只有当本轮工具集中**实际包含** Local 计算机工具，并且这些工具处于激活状态时，才会触发策略查询：

```python
if (
    tools
    and any(
        (tool := tools.get_tool(name)) is not None and tool.active
        for name in (
            "astrbot_execute_shell",
            "astrbot_shell_session",
            "astrbot_execute_python",
        )
    )
    and is_local_runtime(run_context)
):
    local_policy = get_local_permission_policy(run_context)
    if (
        local_policy.allow_execution
        and not local_policy.allow_network
        and LOCAL_NETWORK_POLICY_NOTICE not in request.system_prompt
    ):
        request.system_prompt += f"\n{LOCAL_NETWORK_POLICY_NOTICE}\n"
```

关键契约：

- 只检查 `astrbot_execute_shell`、`astrbot_shell_session`、`astrbot_execute_python` 这三个内建 Local 工具名。
- 如果工具在 `ToolSet` 中不存在（`tools.get_tool(name) is None`），或存在但 `tool.active == False`，则不触发策略查询与注入。
- `is_local_runtime(run_context)` 必须为真；`sandbox` 与 `none` 运行时都不会注入。
- 与 `build_main_agent` 一致：注入前检查 `LOCAL_NETWORK_POLICY_NOTICE not in request.system_prompt`，避免重复。

## run_context 复用

在 `Context.tool_loop_agent(...)` 中，`AgentContextWrapper` 被提取为共享变量 `run_context`：

```python
run_context = AgentContextWrapper(
    context=agent_context,
    tool_call_timeout=tool_call_timeout,
)
# ... 可能基于 run_context 做策略查询与 system_prompt 注入 ...
await agent_runner.reset(
    provider=prov,
    request=request,
    run_context=run_context,
    tool_executor=tool_executor,
    agent_hooks=agent_hooks,
    streaming=streaming,
)
```

因此策略查询使用的上下文与实际传入 runner 的上下文是同一对象，避免因重复构造 `AgentContextWrapper` 而造成状态不一致。

## 工具返回值的简化

网络策略提示已不再通过工具调用结果传播。该变更在以下位置移除了返回值中的策略提示：

- `astrbot_execute_python`（`LocalPythonTool.call`）：不再把 `LOCAL_NETWORK_POLICY_NOTICE` 作为 `mcp.types.TextContent` 插入返回内容前，也不再在异常信息前加策略前缀。异常回退统一为：

  ```python
  return f"Error executing code: {str(e)}"
  ```

- `astrbot_execute_shell`（`LocalExecuteShellTool.call`）：不再使用 `policy_notice` 变量，也不再向 JSON 结果中写入 `policy_notice` 字段。异常回退统一为：

  ```python
  return f"Error executing command: {detail}"
  ```

因此工具返回值现在只承载业务输出或错误信息，网络策略不再混入结果。

## 与 Agent 数据流的关系

调整后的数据流为：

1. 上层通过 `build_main_agent(...)` 或 `Context.tool_loop_agent(...)` 构造 Agent 请求。
2. Agent 构建阶段判断 `computer_use_runtime` 是否为 `local`。
3. 如果 `tool_loop_agent` 路径，其工具集包含激活的 `astrbot_execute_shell`、`astrbot_shell_session` 或 `astrbot_execute_python` 时才继续判断。
4. 调用 `get_local_permission_policy(run_context)` 获取当前角色的 `allow_execution` 与 `allow_network`。
5. 满足 `allow_execution=True` 且 `allow_network=False` 时，把 `LOCAL_NETWORK_POLICY_NOTICE` 追加到 `system_prompt`。
6. 工具执行阶段不再向 LLM 返回网络策略提示，工具结果仅包含命令输出或错误字符串。

该设计把“策略约束”从“事后工具结果”迁移到“事前系统提示”：LLM 在生成工具调用前就能得知本地网络受限，从而降低生成会立即失败的网络访问代码的概率，并避免每次工具调用都重复承载策略说明。

## 变更影响分析

- **策略提示归属从 Tool 结果迁移到 system_prompt**：AI 开发者在排查 Local runtime 网络受限行为时，应检查 `system_prompt` 中是否出现 `LOCAL_NETWORK_POLICY_NOTICE`，而不再依赖工具结果前缀或 `policy_notice` 字段。
- **工具结果字段 `policy_notice` 已移除**：任何读取 `astrbot_execute_shell` 返回 JSON 中 `policy_notice` 字段的旧逻辑将不再生效。解析该工具 JSON 结果时不应再期待该字段。
- **错误 info 更纯净**：`astrbot_execute_python` 的执行异常现在统一返回 `Error executing code: ...`；`astrbot_execute_shell` 的异常统一返回 `Error executing command: ...`。不再在错误消息前拼接网络策略文本。
- **注入条件严格依赖 allow_execution 与 allow_network**：只有“允许执行且禁止网络”时才注入。若角色被禁止执行（`allow_execution=False`）或允许网络（`allow_network=True`），`system_prompt` 中不会出现该提示。
- **tool_loop_agent 需要工具集实际包含 Local 工具**：仅当 `ToolSet` 中的 `astrbot_execute_shell`、`astrbot_shell_session`、`astrbot_execute_python` 至少一个存在且 `active=True` 时才会触发策略注入。仅“运行环境是 local”不足以触发提示注入，避免在无 Local 工具可用时向模型写入无关策略。
- **幂等注入**：`build_main_agent` 与 `Context.tool_loop_agent` 都检查 `LOCAL_NETWORK_POLICY_NOTICE not in system_prompt`，因此调用方若已写入该提示，Agent 层不会重复追加。
- **不影响 sandbox / none 运行时**：`computer_use_runtime != "local"` 时，两条注入路径都不会执行。Sandbox 工具与 Local 工具的策略行为保持分离。
- **权限判定与调用方事件解耦**：在 `tool_loop_agent` 中，策略查询使用 `run_context` 内的 `astr_agent_ctx`（即 `agent_context`），而不是调用方传入的 `event`。当插件显式传入 `agent_context` 时，`get_config(umo=...)` 与权限查询会以 `agent_context.event.unified_msg_origin` 为准。
- **插件 API 签名不变**：`Context.tool_loop_agent(...)`、`build_main_agent(...)`、`FunctionTool` 的签名均未变化。此变更只改变 Agent 构建阶段对 `system_prompt` 的增强，以及 Local 工具返回值的内容。
- **不要依赖工具结果做策略提示**：生成插件代码时，若需要在工具调用后向用户展示网络受限信息，应依赖 `system_prompt` 的注入行为，或自行读取 Local 权限策略，而不是从工具返回文本中提取提示。
