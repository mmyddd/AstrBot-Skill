---
title: Gemini 原生工具与函数工具共存配置 (Gemini Native Tools & Function Tools)
type: improvement
status: stable
last_updated: 2026-10-31
related_base: agent/Invoke-llm.md
---

## 概述

AstrBot 的默认配置模板（`astrbot/core/config/default.py`）与 Dashboard 配置元数据 i18n 文件（`config-metadata.json`，已覆盖 `zh-CN`、`en-US`、`ja-JP`、`ru-RU`）中，Gemini Provider 相关开关的提示文案已修正。

涉及配置项：

- `gm_native_search`：启用原生搜索功能（native search / grounding）。
- `gm_native_coderunner`：启用原生代码执行器（native code runner）。
- `gm_url_context`：启用 URL 上下文功能（URL context）。
- `gm_safety_settings`：安全过滤器（本次未改变其语义，仅作为同一配置组上下文出现）。

核心契约变化是：这三个原生工具开关**不再表述为「启用后所有函数工具将全部失效」**。新的稳定表述为：只有 Gemini 3 及以上模型支持将它们与函数工具（`FunctionTool` / ToolSet）同时使用；并且 Gemini 3 及以上模型的原生搜索仅限付费层级。

## 配置契约（Provider 层）

这三个开关属于 Gemini Chat Provider 的配置字段，语义如下：

| 配置项 | 作用 | 与函数工具的兼容性表述 |
| :--- | :--- | :--- |
| `gm_native_search` | 开启 Gemini 原生搜索 | 仅 Gemini 3 及以上模型支持与函数工具同时使用；Gemini 3 及以上模型的原生搜索仅限付费层级，计费规则以官方文档为准。 |
| `gm_native_coderunner` | 开启 Gemini 原生代码执行器 | 仅 Gemini 3 及以上模型支持与函数工具同时使用。 |
| `gm_url_context` | 开启 URL 上下文 | 仅 Gemini 3 及以上模型支持与函数工具同时使用。 |

旧的提示文案（"启用后所有函数工具将全部失效"）已被移除。这意味着在文档叙述层面，原生工具与函数工具的关系从「互斥」修正为「按模型版本分层的条件共存」。

## 与 Tool / ToolSet 数据流的关系

插件侧注册工具的方式不变，仍是：

```python
from astrbot.api import FunctionTool, ToolSet

self.context.add_llm_tools(MyTool())

llm_resp = await self.context.tool_loop_agent(
    event=event,
    chat_provider_id=await self.context.get_current_chat_provider_id(event.unified_msg_origin),
    prompt="...",
    tools=ToolSet([MyTool()]),
)
```

变化点仅在于对 Provider 行为的描述：当会话使用 Gemini 3 及以上模型时，开启 `gm_native_search` / `gm_native_coderunner` / `gm_url_context` 并不必然导致 ToolSet 中的工具在请求中失效。因此插件不需要仅仅因为用户启用了原生搜索/代码执行/URL 上下文，就自行裁剪 `ToolSet` 或禁用 `add_llm_tools(...)` 注册的工具。

## 变更影响分析

- **旧结论需修正**：AI 助手在分析 Gemini Provider 配置时，不应再输出「开启 `gm_native_search` 会导致所有函数工具失效」这一结论。该结论属于过时提示文案，已被本次修正覆盖。
- **版本条件必须显式表达**：新的兼容性表述带有明确前置条件——Gemini 3 及以上。对于 Gemini 3 以下的模型，原生工具与函数工具是否可共存不由本次文案给出保证，生成建议时应保留模型版本前提，不要泛化为「任何 Gemini 模型都可以共存」。
- **计费提示只针对原生搜索**：`gm_native_search` 的提示额外注明「Gemini 3 及以上模型的原生搜索仅限付费层级」，而 `gm_native_coderunner` 与 `gm_url_context` 的提示只包含模型版本条件，不包含付费层级条件。生成配置说明时不要把这三种工具的能力/计费条件合并成同一条规则。
- **文案变更不等于运行时强制逻辑变更**：本次改动仅涉及配置模板提示文本与多语言元数据，`gm_*` 开关在 Provider 请求构造中的实际注入/裁剪逻辑不在本次变更范围内。排查「工具没被调用」问题时，仍应以 Provider 实现与模型实际能力为准，而不是仅依据提示文案。
- **多语言一致性**：`zh-CN`、`en-US`、`ja-JP`、`ru-RU` 四套 `features/config-metadata.json` 与 `astrbot/core/config/default.py` 的 hint 同步更新。判断配置项语义时应以配置模板（`default.py`）为准，i18n 文件只是同一语义的本地化表达。
- **对插件作者的最佳实践**：使用 `context.add_llm_tools(...)` 或 `tool_loop_agent(..., tools=ToolSet([...]))` 时，无需为 Gemini 原生搜索/代码执行/URL 上下文编写「禁用工具」分支；若需要兼容低版本 Gemini 模型，应将模型版本作为前置校验条件，而不是无条件关闭工具集。
- **未新增对外 API**：本次变更没有引入新的装饰器、Message Chain 组件、`Context` 方法、Provider 接口或 Platform Adapter 接口，也没有新增 `FunctionTool` 定义方式。