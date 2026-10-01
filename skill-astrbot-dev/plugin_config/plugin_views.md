---
title: 插件可视化视图 (Plugin Views)
type: feature
status: stable
last_updated: 2026-10-01
related_base: plugin_config/schema.md
---

## 概述

AstrBot 将插件系统原先的 **Pages** 概念统一更名为 **Views（插件可视化视图）**。插件可以通过 `views/` 目录向 Dashboard 暴露自定义前端页面，`pages` 在所有对外契约位上作为**向后兼容别名**保留。

更名后采用「新名优先 + 旧名别名」的双轨策略，覆盖元数据字段、目录名、i18n key、HTTP 路由、Bridge 全局对象和 Dashboard 前端路由。旧插件无需改动代码即可继续正常工作。

## 插件侧对外契约

### 1. 元数据字段（`metadata.yaml`）

- `StarMetadata.pages: list[dict]` 已重命名为 `StarMetadata.views: list[dict]`，语义为「插件注册的 Views 元数据」。
- `_load_plugin_metadata()` 采用优先级解析：
  1. `metadata["views"]`（若为 list）
  2. 否则 `metadata["pages"]`（若为 list）
  3. 否则 `[]`
- `load()` 重载路径同步改为 `metadata.views = metadata_yaml.views`。

插件作者应优先在 `metadata.yaml` 中使用 `views:`，`pages:` 仅作为兼容别名保留。

### 2. 视图目录（`views/` vs `pages/`）

- 原常量 `PLUGIN_PAGE_ROOT_DIR_NAME = "pages"` 改为有序元组 `PLUGIN_PAGE_ROOT_DIR_NAMES = ("views", "pages")`，`views` 优先。
- `resolve_plugin_pages_root()` 改为**按序轮询**候选目录名，返回首个存在的 `views` 或 `pages` 目录；两者均不存在时抛出：
  ```python
  FileNotFoundError("Plugin views root directory does not exist")
  ```
  目录查找语义已从「单一路径强校验」改为「有序候选轮询」。
- `__all__` 导出相应更新为 `PLUGIN_PAGE_ROOT_DIR_NAMES`。

因此，插件目录结构同时支持：

```text
astrbot_plugin_x/
├─ views/
│  └─ my-view/
│     └─ index.html     # 推荐
└─ pages/                # 兼容别名，views 优先
```

### 3. i18n key 链

`build_initial_context()` 中的标题 i18n 查找链为：

1. `views.{name}.title`
2. `pages.{name}.title`
3. 回退到裸 `page_name`

这与顶层 i18n 的通用「新名优先 + 旧名别名」策略一致。

### 4. Bridge 全局对象

- 前端 Bridge 全局对象由 `window.AstrBotPluginPage` 改为 **`window.AstrBotPluginView`**。
- `serve_bridge_sdk()` 注入脚本由 `window.AstrBotPluginPage?.__setInitialContext(...)` 改为 `window.AstrBotPluginView?.__setInitialContext(...)`。
- 同时新增兼容别名：
  ```js
  window.AstrBotPluginPage = window.AstrBotPluginView
  ```

插件前端代码应优先使用 `window.AstrBotPluginView`，老代码使用 `window.AstrBotPluginPage` 仍可继续工作。

## Dashboard HTTP API 路由

v1 API 端点全部采用**双路由注册**，新旧并存：

| 旧路由 | 新增别名路由 |
|---|---|
| `GET /plugins/pages` | `GET /plugins/views` |
| `GET /plugins/page` | `GET /plugins/view` |
| `GET /plugins/page/assets` | `GET /plugins/view/assets` |
| `GET /plugins/{plugin_id}/pages` | `GET /plugins/{plugin_id}/views` |
| `GET /plugins/{plugin_id}/pages/{page_name}` | `GET /plugins/{plugin_id}/views/{page_name}` |
| `GET /plugins/{plugin_id}/pages/{page_name}/assets/{asset_path:path}` | `GET /plugins/{plugin_id}/views/{page_name}/assets/{asset_path:path}` |

这意味着基于旧端点的集成不会失效，但新代码应使用 `views` 系列路径。

## 内部命名与层级链路

完整数据流如下：

```text
插件元数据 YAML
  └─ views(优先) / pages(别名) → StarMetadata.views
        └─ PluginPageService.resolve_plugin_pages_root()
              └─ 目录查找顺序: ("views", "pages")
                    └─ i18n 标题: views.* → pages.* → 裸名
                          └─ Bridge 全局对象: AstrBotPluginView (AstrBotPluginPage 为别名)
                                └─ API 路由: /views/* 与 /pages/* 双注册
```

**注意**：术语对齐是**部分**的。`PluginPageService`、`PluginPage`、`PluginPageContentPayload` 等**类名未改**，仅元数据字段、目录常量、HTTP 路由和 Bridge 全局对象改名。AI 消费此文档时，不应假设类名也已重命名。

## 插件作者最佳实践

- 在 `metadata.yaml` 中使用 `views:`；不要只依赖 `pages:`。
- 将可视化页面放在插件目录的 `views/` 下；`pages/` 只能作为迁移期兼容目录。
- 前端代码优先使用 `window.AstrBotPluginView`；使用 `window.AstrBotPluginPage` 的代码应逐步迁移。
- i18n 中优先使用 `views.<name>.*` 键；`pages.<name>.*` 作为回退。
- HTTP 集成优先调用 `/plugins/.../views/...`；旧 `/plugins/.../pages/...` 端点未删除但为兼容路径。

## 变更影响分析

- **本次更名为纯兼容重构，不对插件签名产生破坏性变更**：元数据字段、目录名、i18n key、API 路由、Bridge 全局对象均采用「新名优先 + 旧名别名」双轨，老插件代码可继续运行。AI 助手在生成新插件时应使用新名（`views` / `AstrBotPluginView`），在修复老插件时可保持旧名。
- **目录解析语义从「单一路径强校验」变为「有序候选轮询」**：`resolve_plugin_pages_root()` 不再因为 `pages` 目录缺失而直接失败，而是按 `("views", "pages")` 顺序返回首个存在目录。两种目录同时存在时，`views` 胜出。AI 助手在排查「插件视图找不到」问题时，应同时检查 `views/` 与 `pages/` 是否存在以及 `views/` 是否为空目录。
- **异常消息已改为 Views 措辞**：`FileNotFoundError("Plugin views root directory does not exist")` 不再含有 "Pages" 字样。依赖字符串匹配旧异常文本的脚本需要更新。
- **类名与 API 名称未变**：`PluginPageService`、`PluginPageContentPayload` 等仍保留 `PluginPage` 前缀。AI 助手不应编造 `PluginViewService` 之类的类名。
- **HTTP 路由与前端路由是两套并行的过渡层**：后端 v1 API 通过双路由注册保留 `/plugins/page(s)*`，插件前端 page 内引用的 `window.AstrBotPluginView` 是新全局对象，`window.AstrBotPluginPage` 是投影别名。AI 生成的前端示例应优先使用新全局对象与新路由。
- **i18n key 生成应遵守三级回退链**：若已存在 `pages.<name>.title` 的老翻译，新代码可继续沿用；但推荐新键位使用 `views.<name>.title`，缺失时系统自动回退到 `pages.<name>.title`，再回退到裸 `page_name`。
- **术语一致性要求提升**：本变更确立了 AstrBot 中「Views」比「Pages」优先的词表。AI 助手在生成 plugin 文档、注释、示例目录时，应该默认使用 `views/` 目录名与 `views:` 元数据字段，并在涉及旧路径时显式注明为兼容别名。
