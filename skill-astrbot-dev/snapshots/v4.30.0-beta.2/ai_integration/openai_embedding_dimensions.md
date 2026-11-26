---
title: OpenAI Embedding Dimensions 配置行为
type: improvement
status: stable
last_updated: 2026-06-24
related_base: agent/Invoke-llm.md
---

## 概述

OpenAI Embedding Provider 在调用嵌入 API 时，`dimensions` 参数现在只会在 Provider 配置中显式提供 `embedding_dimensions` 时传入。

此前实现会在未配置 `embedding_dimensions` 时仍向嵌入接口传入默认维度 `1024`。这会导致不支持 `dimensions` 参数的嵌入模型在请求阶段返回错误。现在，未显式配置维度时，请求不会携带 `dimensions` 参数，让模型使用自身默认输出维度。

该变更影响的是 Embedding Provider 的请求构造逻辑，不改变插件调用 Provider、Message Chain、Adapter 或 Event Loop 的方式。

## 核心行为

相关实现位于：

```python
astrbot/core/provider/sources/openai_embedding_source.py
```

OpenAI Embedding Source 暴露的嵌入调用方法仍为：

```python
async def get_embedding(self, text: str) -> list[float]
async def get_embeddings(self, text: list[str]) -> list[list[float]]
```

两者在请求前都会调用内部方法构建可选参数：

```python
def _embedding_kwargs(self) -> dict
```

`_embedding_kwargs()` 的稳定行为为：

1. 当 `self.provider_config` 中不存在 `embedding_dimensions` 时，返回空字典。
2. 当 `embedding_dimensions` 存在且可以转换为整数时，返回：

```python
{"dimensions": int(self.provider_config["embedding_dimensions"])}
```

3. 当 `embedding_dimensions` 存在但不是合法整数时，记录 warning，并忽略该配置，不向请求传入 `dimensions`。

因此，实际请求形态从“总是传入 dimensions”变为“仅在显式配置且合法时传入 dimensions”：

```python
embedding = await self.client.embeddings.create(
    input=text,
    model=self.model,
    **kwargs,
)
```

批量嵌入同样遵循相同逻辑：

```python
embeddings = await self.client.embeddings.create(
    input=text,
    model=self.model,
    **kwargs,
)
```

## `embedding_dimensions` 配置契约

`embedding_dimensions` 是 OpenAI Embedding Provider 配置中的可选字段。

- 未配置：不发送 `dimensions` 参数。
- 配置为合法整数：发送 `dimensions=<该整数>`。
- 配置为非法值：记录 warning，并按未配置处理。

非法配置示例包括无法被 `int()` 转换的字符串、空值或类型不匹配的值。非法配置不会中断 `get_embedding()` 或 `get_embeddings()` 的请求构造流程，但会导致 `dimensions` 被忽略。

## `get_dim()` 返回值变化

`get_dim()` 用于读取当前 Embedding Provider 的向量维度配置：

```python
def get_dim(self) -> int
```

新的返回规则为：

1. 如果 `embedding_dimensions` 显式存在且是合法整数，返回该整数。
2. 如果 `embedding_dimensions` 不存在，返回 `0`。
3. 如果 `embedding_dimensions` 存在但不是合法整数，记录 warning，并返回 `0`。

`0` 在这里表示“Provider 配置没有提供可用的显式维度”，而不是表示嵌入模型真实输出 0 维向量。调用方如果依赖 `get_dim()` 判断向量库维度，应把 `0` 视为未知或未指定维度。

## 与 Provider 数据流的关系

该变更位于 Provider 层的 Embedding Source 请求构造阶段。数据流如下：

1. 上层能力调用 Embedding Provider 的 `get_embedding(text)` 或 `get_embeddings(list[str])`。
2. OpenAI Embedding Source 读取自身 `provider_config`。
3. `_embedding_kwargs()` 检查 `embedding_dimensions` 是否显式存在且可转换为整数。
4. 只有在合法配置存在时，才把 `dimensions` 放入 OpenAI embeddings API 请求。
5. 模型返回 embedding 向量后，方法继续返回 `list[float]` 或 `list[list[float]]`。

该设计降低了 AstrBot Provider 配置与具体嵌入模型 API 能力之间的耦合。对于不支持可变维度的模型，AstrBot 不再强制发送多余参数；对于支持自定义维度的模型，用户仍可通过显式配置 `embedding_dimensions` 启用该能力。

## 变更影响分析

- **Embedding Provider 兼容性提升**：不支持 `dimensions` 参数的嵌入模型不会再因为 AstrBot 默认传入 `1024` 而失败。AI 开发者在分析嵌入请求错误时，应先检查是否显式配置了 `embedding_dimensions`。
- **`embedding_dimensions` 不再有隐式默认值语义**：未配置时不会自动等价于 `1024`。如果向量库、知识库或检索逻辑需要固定维度，应由配置或模型实际输出结果明确确定。
- **`get_dim()` 的 `0` 表示未知维度**：AI 助手生成 Provider 或知识库相关代码时，不应把 `get_dim() == 0` 理解为真实向量维度，而应理解为“未显式配置有效维度”。
- **非法维度配置会被忽略而不是抛出异常**：当 `embedding_dimensions` 无法转换为整数时，系统记录 warning 并继续构造请求。这避免了配置错误直接中断 Provider 初始化或调用，但也意味着实际请求可能使用模型默认维度。
- **单条与批量嵌入行为保持一致**：`get_embedding()` 和 `get_embeddings()` 都通过 `_embedding_kwargs()` 构造请求参数，因此不会出现单条请求传参、批量请求不传参的不一致情况。
- **插件侧调用方式不变**：插件作者仍通过 AstrBot 的 Provider 抽象使用 Embedding 能力，不需要改动 Message Chain、AstrMessageEvent、Context 或 Adapter 相关代码。该变更只影响 Embedding Provider 与外部嵌入 API 之间的参数传递契约。
