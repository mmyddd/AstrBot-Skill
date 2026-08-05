---
title: 插件安装目录与 metadata.yaml name 校验
type: improvement
status: stable
last_updated: 2026-06-24
related_base: plugin-development-workflow.md
---

## 概述

AstrBot 插件安装流程现在以插件包内 `metadata.yaml` 的 `name` 字段作为最终插件目录名，并在安装阶段校验该名称是否可作为 Python 模块名被 `importlib` 加载。

这影响从 Git 仓库安装和从 zip 文件上传安装两条路径。插件作者不能再假设插件安装目录一定来自 GitHub 仓库名、zip 文件名、`-master` / `-main` 后缀清洗结果或小写化结果；最终目录会以 `metadata.yaml` 中的 `name` 为准。

## 核心契约

插件包必须包含有效的 `metadata.yaml`，且其中 `name` 字段必须满足以下条件：

1. `name` 必须存在。
2. `name` 必须是非空字符串，去除首尾空白后仍然非空。
3. `name` 不能包含路径分隔符 `/` 或 `\\`。
4. `name` 必须是合法 Python 标识符，即满足 `str.isidentifier()`。
5. `name` 不能是 Python 关键字。

不满足上述条件时，安装流程会抛出异常，插件不会被正常安装为可加载模块。

## 关键方法

新增的安装名称处理逻辑集中在 `PluginManager`：

```python
PluginManager._normalize_plugin_dir_name(plugin_name: str) -> str
PluginManager._validate_importable_name(plugin_name: str) -> None
PluginManager._get_plugin_dir_name_from_metadata(plugin_path: str) -> str
```

其中：

- `_normalize_plugin_dir_name(...)` 当前只执行 `strip()`，用于去除 `metadata.yaml` 中 `name` 的首尾空白。
- `_validate_importable_name(...)` 负责拒绝路径分隔符、非法 Python 标识符和 Python 关键字。
- `_get_plugin_dir_name_from_metadata(...)` 负责读取插件目录下的 `metadata.yaml`，提取 `name`，完成规范化与校验，并返回最终插件目录名。

## Git 仓库安装行为

从 Git 仓库安装插件时，流程会先根据仓库 URL 解析并格式化仓库名，用于检测仓库对应目录是否已经存在：

```python
_, repo_name, _ = self.updator.parse_github_url(repo_url)
repo_name = self.updator.format_name(repo_name)
plugin_path = os.path.join(self.plugin_store_path, repo_name)
```

如果该仓库目录已经存在，安装会提前失败，避免重复下载或覆盖已有目录。

仓库下载完成后，AstrBot 会读取下载结果中的 `metadata.yaml`，以 `name` 字段计算最终目录名。如果最终目录名与下载目录不同，则会将目录重命名为 `metadata.yaml` 中的 `name`。如果目标目录已经存在，安装失败。

因此，插件作者发布 Git 仓库插件时，应保证：

```yaml
name: astrbot_plugin_example
```

对应的值既是稳定插件标识，也是可作为本地插件目录和 Python 模块导入路径的名称。

## zip 上传安装行为

从 zip 文件上传安装插件时，AstrBot 不再直接使用 zip 文件名推导最终插件目录。新的流程是：

1. 在插件存储目录下创建临时解压目录，前缀为 `plugin_upload_`。
2. 将 zip 内容解压到该临时目录。
3. 读取临时目录中的 `metadata.yaml`。
4. 使用 `metadata.yaml` 的 `name` 字段作为最终插件目录名。
5. 如果目标目录已存在，则安装失败。
6. 如果目标目录不存在，则将临时目录重命名为最终插件目录。
7. 安装失败或目录迁移后，会尝试清理不再使用的临时解压目录。

这意味着 zip 文件名不再是插件导入名称的权威来源。插件作者应把 `metadata.yaml.name` 视为插件安装和加载的稳定标识。

## 与插件加载模型的关系

AstrBot 的插件是 `Star`，由插件管理器加载、卸载和重载。插件目录名会参与模块导入路径构造，因此目录名必须能被 Python 导入系统识别。此次变更把“插件身份”从外部包名收敛到 `metadata.yaml.name`，减少了 Git 仓库名、zip 文件名和插件元数据之间的不一致。

该设计降低了插件分发方式与运行时加载方式之间的耦合：无论插件来自 Git 仓库还是 zip 上传，只要包内 `metadata.yaml.name` 相同且合法，AstrBot 都会尝试使用同一套目录命名规则进行安装。

## 插件作者最佳实践

推荐插件 `metadata.yaml` 使用稳定、合法、唯一的模块式名称：

```yaml
name: astrbot_plugin_weather
```

避免以下写法：

```yaml
name: astrbot-plugin-weather   # 包含连字符，不是合法 Python 标识符
name: my/plugin                # 包含路径分隔符
name: class                    # Python 关键字
name: ""                       # 空名称
```

插件目录、仓库名和 zip 文件名可以不同，但为了降低排障成本，建议保持三者尽量一致，尤其是公开发布的插件。

## 变更影响分析

- **插件安装标识更权威**：AI 开发者在生成或检查插件项目时，应优先校验 `metadata.yaml.name`，而不是只检查目录名或 zip 文件名。
- **非法名称会在安装阶段失败**：包含 `-`、空格、路径分隔符或 Python 关键字的插件名不再适合作为可安装插件名。生成插件模板时应使用下划线命名，例如 `astrbot_plugin_xxx`。
- **zip 文件名不再决定最终目录**：上传 `weather-main.zip` 并不意味着最终目录是 `weather` 或 `weather-main`；最终目录由包内 `metadata.yaml.name` 决定。
- **目录冲突会阻止安装**：如果根据仓库名或 `metadata.yaml.name` 计算出的目录已经存在，安装会失败。AI 助手在排查安装失败时，应同时检查仓库目录名和元数据目录名是否与已有插件目录冲突。
- **跨分发方式行为更一致**：Git 安装和 zip 上传都收敛到 `metadata.yaml.name`，插件作者可以把该字段作为插件运行时身份、目录命名和导入合法性的核心契约。
- **不要把插件名称写成展示名**：`metadata.yaml.name` 是机器可读标识，不应包含中文、空格、连字符或展示性文本。面向用户的名称应使用 `display_name` 等元数据字段。 
