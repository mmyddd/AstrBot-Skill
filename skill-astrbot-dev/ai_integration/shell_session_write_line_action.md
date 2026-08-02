---
title: Shell Session 工具新增 write_line 操作
type: improvement
status: stable
last_updated: 2026-08-02
related_base: agent/offical-tool-list/tools.md
---

## 概述

`astrbot_shell_session` 工具（`ShellSessionTool`）新增一个 `write_line` 操作，用于向托管的 shell 会话写入一行完整的输入。与 `write` 操作逐字发送文本不同，`write_line` 会自动在文本末尾附加一个真实的换行符（LF, `\n`）。这使面向行的交互式程序（如 REPL、逐行输入的命令）能够收到完整的行结束信号。

该变更同时更新了本地模式提示（`_build_local_mode_prompt()`），引导 Agent 对面向行的程序使用 `write_line` 操作，并保持“普通长时间运行的命令不需要 `&`、`nohup` 或分离包装”的既有指导不变。

## 核心行为

工具名称：`astrbot_shell_session`。

操作枚举更新为：

```python
["list", "poll", "write", "write_line", "interrupt", "terminate"]
```

- `write`: 逐字发送 `chars` 字段中的文本，不附加额外字符。
- `write_line`: 向 `write_session(...)` 传递 `chars` 时，自动格式化为 `f"{chars}\n"`，保证会话接收到真实的换行。

工具描述更新为：

> List, poll, write raw text or complete lines to, interrupt, or terminate managed shell sessions.

`chars` 参数描述更新为：

> Text sent verbatim by write. For write_line, provide one line without a line ending; a real LF is appended automatically.

本地模式提示（`_build_local_mode_prompt()`）同步更新，明确指示：

> Use its `write_line` action for line-oriented programs so the session receives a real line feed.

## 深入分析

该变更位于 Shell Session 工具的执行分支，核心改动为：

```python
elif action in {"write", "write_line"}:
    result = await sb.shell.write_session(
        owner_id=owner_id,
        requester_id=requester_id,
        requester_is_admin=requester_is_admin,
        session_id=session_id,
        chars=f"{chars}\n" if action == "write_line" else chars,
    )
```

对于 AI 开发者，这意味着：

- 当 Agent 需要向托管会话发送交互式命令的输入（例如提示用户输入、确认、选择等）时，应优先使用 `write_line`，确保程序正确接收回车。
- 对二进制流、部分输入或逐字符回显场景，仍应使用 `write`，它保持逐字发送语义，不添加任何额外字节。
- 该变更没有改变会话隔离规则：会话仍被隔离到当前会话和发送者，管理员可以管理会话中的所有会话。`write_line` 不会绕过身份校验或会话权限检查。

该变更属于核心工具链的对外契约变化：`astrbot_shell_session` 的 JSON Schema 直接暴露给 LLM，因此 AI 调用方可以在工具参数中合法选择 `write_line`。插件作者无需额外适配即可受益，因为工具 schema 由 AstrBot 核心自动维护。

## 变更影响分析

- **AI 工具契约变化**：`astrbot_shell_session` 的 `action` 枚举新增 `write_line`。任何根据该 schema 生成工具调用参数的 AI 助手都可以在需要完整行输入时选择 `write_line`，而不需要手动在 `chars` 中追加 `\n` 或依赖 `write` 的隐式行为。
- **`chars` 语义更明确**：`chars` 字段同时服务 `write` 与 `write_line`。对于 `write_line`，`chars` 不应包含行尾；工具会自动追加 LF。AI 助手在生成参数时应避免双重添加换行，否则可能出现空行或连续换行。
- **不改变既有操作**：`list`、`poll`、`write`、`interrupt`、`terminate` 的行为保持不变。已有依赖 `write` 逐字发送的 Agent 逻辑不受影响。
- **面向行程序的输入可靠性提升**：以前 Agent 若使用 `write` 发送交互式命令的文本，可能因缺少换行而让程序停留在等待输入状态。现在 `write_line` 提供了显式的行结束语义，减少了交互式 shell 会话的挂起问题。
- **对插件开发的影响**：插件作者不需要修改代码即可通过 `self.context.tool_loop_agent(...)` 使用该工具。若插件自定义了类似的 shell 会话工具，应参考该模式为面向行的程序提供 `write_line` 语义，并在工具描述中清晰区分 `write` 与 `write_line` 的输入差异。
- **文档提示同步更新**：本地模式提示（`_build_local_mode_prompt()`）已经把 `write_line` 纳入推荐用法。AI 助手在分析 Local Computer 或 Sandbox 的 Shell 会话工具时，应预期该工具支持面向行的写入操作，并在生成工具调用时优先使用 `write_line` 处理需要换行的输入。
