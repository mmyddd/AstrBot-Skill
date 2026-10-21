---
title: Gemini Tool Schema 枚举参数转换 (Enum Conversion)
type: improvement
status: stable
last_updated: 2026-10-21
related_base: agent/register tools.md
---

## 概述

在把 AstrBot 的 `FunctionTool` 参数 JSON Schema 转换为 Gemini（Google GenAI）可接受的 function declaration schema 时，`convert_schema(schema: dict) -> dict` 现在会重写 `enum` 字段，避免生成 Gemini 不接受的非字符串枚举。

背景约束是 Gemini 侧的两个硬性规则：Gemini API 只允许在 `STRING` 类型字段上使用 `enum`，且 `google.genai` 的 `Schema.enum` 只接受字符串。此前若插件/MCP 工具声明了整数或布尔枚举参数（例如 FastMCP 为 `Literal[10, 20, 50]` 生成的 `{"type": "integer", "enum": [10, 20, 50]}`），在构造 Gemini 请求时会在发出前抛出 `ValidationError`。

本变更只修改 `ToolSet.google_schema()` 产出的 schema，不改变 `FunctionTool.parameters` 中保存的原始 JSON Schema，也不改变 OpenAI 侧的 schema 转换路径。

## 转换规则

实现位于 `astrbot/core/agent/tool.py` 的 `convert_schema(...)`，在为每个字段构造 `result` 之后、处理 `properties` 之前执行：

1. 若 `result` 中存在 `enum`，先 
2. 过滤掉其中的 `None`，得到有效取值列表 `values`。
3. 若原始 `enum` 中包含 `None`，设置 `result["nullable"] = True`。
4. 若 `result["type"] == "string"`：保留 `enum` 语义，把每个取值转换为字符串——字符串原样保留，其他类型使用 `json.dumps(v)` 序列化。
5. 若 `result["type"]` 不是 `"string"`：完全移除 `enum`，并把允许值以 `Allowed values: v1, v2, ...` 的形式追加到该字段的 `description` 末尾（原 `description` 为空时不会产生多余前导换行，因为结果会执行 `.strip()`）。

稳定行为对照（来自单元测试期望）：

- 输入 `{"type": "integer", "description": "Items per page.", "enum": [10, 20, 50]}` → 输出 `{"type": "integer", "description": "Items per page.\nAllowed values: 10, 20, 50"}`。
- 输入 `{"type": "boolean", "enum": [True]}` → 输出 `{"type": "boolean", "description": "Allowed values: true"}`。
- 输入 `{"type": ["string", "null"], "enum": ["asc", "desc", None]}` → 输出 `{"type": "string", "enum": ["asc", "desc"], "nullable": True}`。

关键设计取舍：非字符串枚举字段仍保留原始 `type`（例如 `integer`），使模型仍需按正确类型传参；枚举约束被降级为描述性提示，由模型自然语言理解来遵守。字符串枚举字段仍保留结构化 `enum` 约束，因此受控词表类参数（如排序方向）在 Gemini 上依然是强约束。

## 数据流位置

该逻辑处于 Tool 定义到 Provider 请求之间的 schema 适配层：

1. 插件通过 `FunctionTool` / dataclass Tool / FastMCP 生成 `parameters`（标准 JSON Schema，可能含非字符串 `enum`）。
2. 工具被加入 `ToolSet([...])`。
3. 调用 `ToolSet([tool]).google_schema()` 时，`convert_schema(...)` 逐字段重建 Gemini 兼容 schema，其中包含本变更的 `enum` 重写。
4. 该 schema 作为 `function_declarations[...]["parameters"]` 交给 `google.genai` 构造 `genai_types.Tool(...)`，再由 Gemini 模型消费。

由于重写针对单个字段的 `result` 构造过程，OpenAI schema 生成路径以及工具注册、`context.add_llm_tools(...)`、`context.tool_loop_agent(...)` 的调用签名均不受影响。

## 与原始 parameters 的关系

`convert_schema(...)` 产出的是一份用于 Provider 请求的副本语义结果：单元测试同时断言，转换后 `tool.parameters["properties"]["page_size"]["enum"]` 仍为 `[10, 20, 50]`。

这意味着：

- 同一份工具定义可以在不同 Provider 之间复用，原始 JSON Schema 不会被 Gemini 适配逻辑污染。
- 插件作者不需要为 Gemini 单独维护一份「字符串枚举」参数定义。
- 若插件在运行时读取 `tool.parameters` 做参数校验或文档展示，看到的仍是原始枚举值与类型。

## 变更影响分析

- **Gemini 工具注册不再因非字符串枚举崩溃**：带 `enum: [10, 20, 50]`、`enum: [true]` 等参数的 `FunctionTool` 在 Gemini Provider 上不再导致构造请求时的 `ValidationError`。AI 开发者应先检查 `enum` 参数类型，再判断工具调用失败是否属于 schema 兼容问题。
- **非字符串枚举的约束强度下降**：整数/布尔枚举在 Gemini 上不再有结构化约束，只是 `description` 中的 `Allowed values: ...` 文本提示。生成插件代码时，若参数取值必须被严格限制，应在工具实现内部做二次校验，而不是只依赖 schema。
- **字段 `description` 会被改写**：非字符串枚举字段的 `description` 末尾会追加以 `Allowed values: ` 开头的行。任何依赖 `description` 精确原文做匹配、去重或提示工程的逻辑，都不应假设描述与源码中书写的一致。
- **枚举中的 `None` 语义被显式化**：`None` 会被从取值列表中移除，并转为 `nullable: True`。不要把 `None` 当作普通枚举值，也不要在字符串枚举里用 `None` 表达「空字符串」之类的业务含义。
- **字符串枚举仍是强约束**：`type: "string"` 的字段继续携带 `enum`（非字符串取值会被 `json.dumps` 转成字符串）。在 Gemini 上，字符串枚举参数的枚举约束依然有效。
- **非字符串枚举的 `type` 不被改写**：`integer` 仍是 `integer`，`boolean` 仍是 `boolean`，模型应发送对应类型的值，而不是字符串。AI 开发者在解释工具参数类型时不要因为看到「允许值列表出现在描述里」就推断该参数变成了字符串参数。
- **仅影响 Gemini schema 覆盖层，不影响工具定义本身**：`FunctionTool.parameters`、OpenAI schema、工具注册与 Tool 执行链路未改变。排查问题时应区分「工具定义错误」与「Gemini schema 适配结果」。
- **最佳实践**：为跨 Provider 兼容性，优先使用字符串枚举（如 `Literal["asc", "desc"]`）表达有限取值；必须使用非字符串枚举时，为字段写好清晰 `description`，并在工具实现中做取值校验。
