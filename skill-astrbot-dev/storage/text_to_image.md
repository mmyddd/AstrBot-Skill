---
title: 文转图 (Text to Image)
type: improvement
status: stable
last_updated: 2026-10-01
related_base: design_standards/visual_utils.md
---

## 概述

`Star.text_to_image` 与 `Star.html_render` 现在支持按会话（UMO）解析配置，并可将 T2I 渲染端点作为**逐调用参数**透传到渲染链路。同时，`Context` 对象新增 `html_renderer` 字段，插件应通过 `self.context.html_renderer` 访问渲染器，全局 `astrbot.core.html_renderer` 已标记为 Deprecated。

该变更影响插件在会话级配置下渲染图片、选择 T2I 模板和指定 T2I 渲染端点的行为。

## 插件 API 变更

### Star.text_to_image

```python
async def text_to_image(
    self,
    text: str,
    return_url: bool = True,
    template_name: str | None = None,
    umo: str | None = None,
) -> str
```

- 新增 `template_name`：显式指定模板，**优先级高于**配置中的 `t2i_active_template`。
- 新增 `umo`：用于通过 `self.context.get_config(umo)` 解析该会话绑定的配置文件；未传入或未绑定时回退默认配置。
- 当 `get_config(umo)` 返回 `None`（例如某些 Context 无默认配置）时，方法内部会回退为 `None` 而非抛异常，渲染器使用默认模板。

### Star.html_render

```python
async def html_render(
    self,
    tmpl: str,
    data: dict,
    return_url: bool = True,
    options: dict | None = None,
    umo: str | None = None,
) -> str
```

- 新增 `umo`，语义同 `text_to_image`。
- 内部辅助方法 `_get_context_config()` 已被删除；端点解析改为直接从解析出的配置读取 `config.get("t2i_endpoint") or None`。

兼容性：旧签名 `text_to_image(text, return_url)` 依旧可用；`template_name`、`umo` 均为可选参数。

## 渲染端点透传

渲染调用链为：

```text
Star.text_to_image / Star.html_render
  -> Context.html_renderer.render_t2i / render_custom_template
    -> NetworkStrategy.render / render_custom_template
```

`render_t2i`、`render_custom_template`、`NetworkStrategy.render`、`NetworkStrategy.render_custom_template` 均新增可选形参：

```python
endpoint: str | None = None
```

端点选择规则：

- `endpoint` 非空：使用 `endpoints = [self._clean_url(endpoint)]`，单点调用、**不进行随机打散**。
- `endpoint` 为空：沿用原有 `self.endpoints` 池并执行 `random.shuffle(...)`，若为空则回退 `BASE_RENDER_URL`。

Pipeline 的 `result_decorate` 阶段在自动文转图时也会传入端点：

```python
endpoint=self.ctx.astrbot_config.get("t2i_endpoint") or None
```

## Context 持有 HtmlRenderer

- `Context` 新增字段 `html_renderer: HtmlRenderer`。
- 由 `core_lifecycle.initialize()` 注入：`self.star_context.html_renderer = html_renderer`。
- 全局变量 `astrbot.core.html_renderer` 仍保留，但已 **Deprecated**；插件应改为使用 `self.context.html_renderer`。

## 配置结构重组（对插件开发者的可见影响）

- `t2i` 配置块（含 `t2i` bool、`t2i_word_threshold`、`t2i_strategy`、`t2i_endpoint`、`t2i_template`、`t2i_active_template`）从原“其他配置”迁移至**系统配置**（`CONFIG_METADATA_3_SYSTEM.system_group.system`）。
- 迁移后按配置档案（config profile）作用域生效，而不是全局固定值。插件通过 `umo` 调用渲染 API 时会读取该会话绑定档案中的 T2I 配置。
- 条件可见性：`t2i_word_threshold`、`t2i_strategy` 新增 `condition: {t2i: True}`；`t2i_endpoint`、`t2i_template` 在原有条件上追加 `t2i: True`。
- `t2i_active_template` 标记为 `invisible`，不再出现在通用渲染器中。

## 与 Message Chain 的关系

本变更不新增任何 Message Chain 组件。渲染结果仍为图片路径或 URL，插件仍应通过 `event.image_result(url)` 或构造 `Comp.Image` 发送：

```python
url = await self.text_to_image(
    "你好",
    umo=event.unified_msg_origin,
    template_name="my_template",
)
yield event.image_result(url)
```

## 变更影响分析

- **端点由全局/配置级下沉为逐调用级参数**：优先级为 `endpoint` 参数 > 配置池 > 默认 `BASE_RENDER_URL`。AI 助手在分析 T2I 请求目标时，应把端点视为可分调用覆盖的入参，而不是固定全局值。
- **多档案（umo）感知**：`Star.text_to_image` 与 `Star.html_render` 现在会依据 `umo` 绑定的配置文件读取模板与端点。生成插件代码时，若希望 T2I 结果随会话配置切换，应显式传入 `umo=event.unified_msg_origin`。
- **`get_config(umo)` 可能为 `None`**：在无默认配置的实现中，`Star.text_to_image` 会降级到 `template_name=None`，不应假设配置一定存在。
- **模板选择优先级明确**：显式 `template_name` 优先于 `t2i_active_template`；未指定时使用配置值，配置缺失时回退默认模板。
- **渲染器访问方式迁移**：应使用 `self.context.html_renderer` 而不是 `from astrbot.core import html_renderer`；后者虽然保留但仍可用，但已被标记 Deprecated。
- **调用无需重启即可切换模板/端点**：模板与端点在每次调用按配置解析并透传，不使用渲染器缓存，因此修改配置后无需重启即可生效（与全局模板缓存语义不同）。
- **T2I 配置位置变化会改变 AI 检索路径**：T2I 配置现已移入系统配置下的配置档案区。AI 助手在检索或生成配置文档时，不应再按“其他配置 / 外观设置”路径查找 `t2i_endpoint`、`t2i_template` 等键。
- **不涉及前端契约**：Dashboard 中 `invisible` 过滤与 Settings 重命名均属于 UI 层，不影响插件 API。
