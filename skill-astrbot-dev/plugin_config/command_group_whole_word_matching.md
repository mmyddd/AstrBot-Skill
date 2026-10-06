---
title: 指令组名称整词匹配行为 (CommandGroupFilter Whole-Word Matching)
type: improvement
status: stable
last_updated: 2026-10-06
related_base: plugin_config/command.md
---

## 概述

`CommandGroupFilter` 的匹配行为已与 `CommandFilter` 对齐：指令组现在按「整词」匹配消息，而不再使用裸 `str.startswith(...)`。这修复了一个长期隐患：当一个指令组名为 `math` 时，旧逻辑会把 `mathematics` 也判为 `math` 组的匹配项。

该修复的直接影响对象是插件作者定义的 `@filter.command_group(name)` 指令组，尤其是当指令组叠加了 `@filter.permission_type(...)` 等访问控制装饰器时，非管理员会因为一条普通私聊消息仅以组名开头而被错误回复「无权限」。

## 核心行为

`CommandGroupFilter` 内部的两个匹配方法被重写为整词语义：

```python
def startswith(self, message_str: str) -> bool
def equals(self, message_str: str) -> bool
```

两个方法都会先对输入消息做规范化：

```python
message_str = re.sub(r"\s+", " ", message_str.strip())
```

即：去除首尾空白，并把任意连续的空白字符（空格、Tab、换行）折叠为单个 ASCII 空格。

`startswith` 的新判定规则为：消息规范化后，与某个完整指令名（`get_complete_command_names()` 返回的名字，包括别名）**完全相等**，或以 `f"{name} "`（名字 + 单个空格）开头。这在语义上等价于「消息第一个整词等于组名」。

`equals` 的新判定规则为：补上同样的空白规范化后，再检查消息是否落在 `get_complete_command_names()` 集合中。这一步是补充性质的重构，确保 `startswith` 与 `equals` 使用一致的规范化规则。

## 嵌套指令组

嵌套指令组（`parent_group` + 子组）同样受益于整词匹配。例如：

```python
parent = CommandGroupFilter("tool")
child = CommandGroupFilter("config", parent_group=parent)
```

- `tool config set a 1` -> 匹配成功。
- `tool configure` -> 不再被误判为 `config` 子组。

当仅输入组名本身、但缺少子指令时（例如 `tool config`、`tool   config`、`tool\tconfig`、`tool\nconfig`），`CommandGroupFilter.filter(...)` 会抛出 `ValueError("参数不足")`，用于触发帮助信息展示。该行为在空白规范化后一致生效，意味着 Tab、换行等分隔符不再导致该分支被跳过。

## 与权限装饰器的交互

当指令组叠加 `@filter.permission_type(filter.PermissionType.ADMIN)` 时，整词匹配带来的关键行为变化是：普通成员发送的「仅以组名开头的普通对话」不再被误判为该组成员的命中。例如：

- 消息 `天气真好啊`（普通私聊聊天）-> 不匹配 `天气` 组，不会被回复权限不足。
- 消息 `天气 设置 北京`（真正的组指令）-> 命中 `天气` 组，非管理员被拒绝。

这使 AstrBot 的权限检查与指令匹配逻辑之间的边界更加清晰：`PermissionTypeFilter` 只对真正命中该指令组的消息生效，而不是对所有以组名开头的噪声文本生效。

## SessionPluginManager 与 WakingCheckStage

指令组的匹配发生在 `WakingCheckStage` 决定事件是否唤醒某个 Handler 的阶段。`WakingCheckStage` 在处理 `AdapterMessageEvent` 事件时会：

1. 通过 `star_handlers_registry.get_handlers_by_event_type(...)` 取得候选 Handler。
2. 通过 `SessionPluginManager.filter_handlers_by_session(...)` 按会话过滤候选插件。
3. 应用 `handler.event_filters` 中注册的过滤器，其中包括 `PermissionTypeFilter` 与 `CommandGroupFilter`。

新的整词匹配收敛了第 3 步中 `CommandGroupFilter` 的命中范围，因此直接减少了第 2 步之后被错误拦截的消息数量。

## 变更影响分析

- **误触发权限拒绝的风险降低**：整词匹配是防止「以组名开头的普通消息被误判为组指令」的关键手段。生成插件代码时，若一个指令组叠加了 `PermissionTypeFilter`，AI 助手不应再假设「以组名开头的私聊消息」等同于组指令调用。
- **`startswith` 的语义与 `CommandFilter` 对齐**：`CommandGroupFilter.startswith` 现在等同「消息首词等于组名/别名」。AI 开发者在生成或审查过滤器逻辑时，应把两者视为同一套匹配契约，而不是旧版 `str.startswith` 语义。
- **空白规范化是双向的**：`startswith` 与 `equals` 都执行 `re.sub(r"\s+", " ", message_str.strip())`。这意味着 `tool\tconfig`、`tool\nconfig`、`tool   config` 与 `tool config` 语义等价。AI 助手在分析消息匹配时应先在概念上完成这层规范化。
- **嵌套组缺少子指令会抛 `ValueError`**：这是 AstrBot 用来展示帮助信息的显式控制流，而不是程序的非预期错误。AI 助手生成嵌套指令组时，不应把该 `ValueError` 视为需要吞掉的异常。
- **不存在模糊前缀匹配的遗留兼容语义**：`mathematics` 不再匹配 `math`，`math123` 也不再匹配 `math`。AI 开发者在生成插件名或指令组名时不需要为了避开前缀冲突而刻意改名；匹配层已保证边界。
- **指令组名不应包含空格**：由于匹配以「名字 + 单空格」作为命令边界，若组名本身含空格，`get_complete_command_names()` 生成的名字会与规范化后的消息难以形成单一整词边界。生成插件代码时应使用无空格、无空白的指令组名，与 `@filter.command` 对指令名的既有约定保持一致。
- **未变更的外部契约**：`@filter.command_group(name)`、`@组名.command("sub")`、`@filter.permission_type(...)`、`CommandGroupFilter(name, alias=..., parent_group=...)` 以及 `WakingCheckStage` 的对外行为均未变化。本变更仅收敛了 `CommandGroupFilter` 对消息文本的匹配范围。
