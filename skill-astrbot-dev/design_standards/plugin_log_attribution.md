---
title: 插件日志归属 (Plugin Log Attribution)
type: improvement
status: stable
last_updated: 2026-11-01
related_base: plugin-development-workflow.md
---

## 概述

AstrBot 的日志记录富化逻辑现在支持在无法从调用栈路径推断插件名时，读取 `LogRecord` 上显式传入的 `plugin_name`。该字段通过标准 `logging` 的 `extra` 参数传入，例如 `logger.error(..., extra={"plugin_name": plugin_name})`。当该字段存在时，记录会被标记为插件来源：`record.plugin_tag = f"[{explicit_plugin_name}]"`、`record.source = "plugin"`、`record.plugin_name = explicit_plugin_name`。

该行为用于框架代码“代插件记录日志”的场景，例如插件加载失败、插件导入 traceback、插件 Handler 或 Hook 执行异常。插件作者通常不需要在每个日志调用中手动传入 `plugin_name`；正常插件路径下的日志仍由路径推断逻辑处理。显式字段只在路径推断没有产生插件名时作为回退归属来源。

## 核心契约

日志记录富化仍先尝试从 `record.pathname` 推断插件名。当路径推断得到插件名时，使用该插件名设置 `plugin_tag`、`source` 与 `plugin_name`。当路径推断没有得到插件名时，进入回退分支：

1. 读取 `explicit_plugin_name = getattr(record, "plugin_name", None)`。
2. 如果 `explicit_plugin_name` 为真，则设置：
   - `record.plugin_tag = f"[{explicit_plugin_name}]"`
   - `record.source = "plugin"`
   - `record.plugin_name = explicit_plugin_name`
3. 否则，根据 `_is_plugin_path(record.pathname)` 设置 `record.plugin_tag` 为 `"[Plug]"` 或 `"[Core]"`，`record.source` 为 `"plugin"` 或 `"core"`，并将 `record.plugin_name` 置为 `None`。

因此，显式 `plugin_name` 不是覆盖路径推断结果的强制重命名；它是在路径无法给出插件名时的补充归属。调用方如果已经位于可被识别为插件路径的代码中，仍会以路径推断出的插件名为准。

## 数据流与调用点

`star_manager.py` 的插件加载流程通过 `logging.LoggerAdapter` 为加载日志绑定 `plugin_name`：

```python
plugin_logger = logging.LoggerAdapter(
    logger, {"plugin_name": root_dir_name}
)
plugin_logger.info("Loading plugin %s ...", root_dir_name)
```

在读取到插件元数据后，适配器会切换为元数据名称：

```python
plugin_logger = logging.LoggerAdapter(
    logger, {"plugin_name": metadata.name or root_dir_name}
)
```

插件导入失败时，traceback 与失败消息都通过该 `plugin_logger` 输出。插件 Handler 与 Hook 的异常日志也通过 `extra={"plugin_name": plugin_name}` 归属到对应插件。对于 `OnPluginLoadedEvent` 的 Hook，日志会使用 `handler_plugin.name if handler_plugin else None` 或 `handler.handler_module_path` 作为 `plugin_name`。

日志富化阶段最终把这些记录统一为带有 `plugin_name` 和 `source="plugin"` 的 `LogRecord`。下游日志消费者可以据此按插件分组、筛选和展示插件故障日志，而不需要从 traceback 文本或文件路径反推插件。

## 与插件开发模型的关系

该变更不新增装饰器、Provider 方法、Message Chain 组件或 Platform Adapter 接口。它不改变 `AstrMessageEvent`、`Context`、`Star` 生命周期或插件加载成功后的行为。变化集中在日志记录的元数据归属：框架在非插件路径代码中代插件输出日志时，可以通过 `extra={"plugin_name": ...}` 让记录进入插件来源分类。

插件作者在普通插件代码中使用 `astrbot.api.logger` 时，通常仍由日志富化逻辑根据插件文件路径识别插件名，不需要额外传 `plugin_name`。如果插件编写了位于插件目录之外的包装层或工具层，并希望这些日志归属到某个插件，可以显式传入 `plugin_name`；但应确保传入的是稳定、可读的插件标识，而不是临时展示文本。

## 变更影响分析

- **插件加载失败日志不再一律归入核心日志**：插件导入失败、加载 traceback 等日志现在可以通过 `plugin_name` 归属到插件。AI 开发者在分析启动失败时，应预期这些日志的 `source` 为 `"plugin"`，并带有插件名，而不是仅看到 `[Core]`。
- **显式 `plugin_name` 只在路径推断失败时生效**：如果日志调用栈路径本身已经能推断出插件名，`extra={"plugin_name": ...}` 不会覆盖该结果。生成代码时不要依赖显式字段去重命名一个已经被路径识别为其他插件的日志。
- **加载早期与后期可能使用不同名称**：插件加载开始时使用 `root_dir_name` 作为 `plugin_name`，读取 `metadata.yaml` 后切换为 `metadata.name or root_dir_name`。如果插件在导入阶段失败、尚未读取元数据，日志会使用目录名；如果失败发生在元数据加载之后，日志会使用元数据名称。排查加载失败时应同时考虑这两个标识。
- **Handler / Hook 异常日志可按插件过滤**：`context_utils.py` 与 `star_request.py` 中的异常日志会携带 `plugin_name`，因此插件 Handler 或 Hook 抛出的异常不再表现为纯核心错误。AI 助手在定位插件异常时，应优先按 `plugin_name` 过滤，而不是只搜索 traceback。
- **`plugin_tag` 格式存在两种来源**：路径推断成功或显式 `plugin_name` 成功时，标签形如 `[<plugin_name>]`；两者都没有时，标签为 `[Plug]` 或 `[Core]`。不要把 `[Plug]` 理解为具体插件名，它只是通用来源标记。
- **`record.plugin_name` 可能为 `None`**：当日志既不在可识别插件路径下，也没有显式 `plugin_name` 时，`record.plugin_name` 为 `None`，`source` 为 `"core"`。日志消费者不应假设每条记录都有插件名。
- **不是新的跨平台消息契约**：该变更不影响 Message Chain、Provider、Adapter、ConversationManager 或 Agent Runner 的调用方式。插件作者不应为了日志归属而修改消息发送或事件处理逻辑。
- **最佳实践是让框架自动归属**：普通插件日志优先依赖路径推断；只有在框架层、加载器、包装器或跨目录工具代码代插件记录日志时，才使用 `extra={"plugin_name": ...}`。这样可避免在插件业务代码中重复维护插件名。