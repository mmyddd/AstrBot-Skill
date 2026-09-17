---
title: 动画帧拼图（GIF/APNG）提示注入
  
type: improvement
status: stable
last_updated: 2026-09-17
related_base: agent/context-management.md
---

## 概述

AstrBot 的图片输入预处理阶段（agent_sub_stages/image_input）会在检测到动图输入（GIF/APNG/多帧 WebP）时，把动画帧拼贴为单张 3x3 网格图片再送入模型。此前该拼贴转换对模型不可见，导致模型把「接触印相表」误读成一张普通静态图（例如把三帧 GIF 理解为“九个相同的角色”）。

本次变更让图片准备阶段显式告知模型：当前输入是动画帧拼图、帧按阅读顺序排列。实现分为两层：

- 工具层：`astrbot/core/utils/media_utils.py` 的 `prepare_model_image(...)` 返回值从 `str | None` 变为 `tuple[str, bool] | None`，第二个元素表示本次结果是否为动画帧拼图（montage），包含缓存命中场景。
- Pipeline 层：`astrbot/core/pipeline/process_stage/method/agent_sub_stages/image_input.py` 的 `prepare_request_images(...)` 收集所有拼图结果，并在本轮请求中追加一条转瞬即逝的 `TextPart` 系统提示。

插件作者仍然使用标准 `Image` 消息组件和 `ProviderRequest` 上下文流程，无需直接调用 `prepare_model_image`。该变更只影响请求进入模型前的上下文构造。

## 核心行为

### `prepare_model_image` 返回值契约

```python
async def prepare_model_image(
    image_ref,
    max_size,
    output_dir: Path,
    quality: int = IMAGE_COMPRESS_DEFAULT_QUALITY,
    montage_max_size: int | None = None,
) -> tuple[str, bool] | None
```

稳定行为：

- 成功时返回 `(path_str, is_montage)`。
- `is_montage` 在内部通过 `_inspect_image(image_bytes)` 得到的帧数判断：`is_montage = frame_count > 1`。
- 当帧数 > 1 时，仍调用 `_extract_animation_montage_sync(...)` 生成拼图字节。
- 缓存命中时仍返回 `is_montage=True`，即拼图属性与工作路径解耦，不会因为命中 cache 而丢失动画语义。
- 可恢复错误或写入失败仍返回 `None`，保持与旧行为一致。

### Pipeline 收集与提示注入

`prepare_request_images(...)` 在遍历图片引用时：

1. 为每个 `ref` 调用 `prepare_model_image(...)`。
2. 解包 `path, is_montage = image`。
3. 对每个成功路径调用 `event.track_temporary_local_file(path)`。
4. 使用 `has_montage |= is_montage` 累积是否出现动画拼图。
5. 在完成所有图片处理后，如果 `has_montage` 为真，则向 `req.extra_user_content_parts` 追加一条 `TextPart`：

```python
req.extra_user_content_parts = [
    *req.extra_user_content_parts,
    TextPart(
        text="<system_notice>\n"
        "The input includes a GIF converted into a single image with frames "
        "in reading order. Treat it as an animation; "
        "do not mention the conversion or frame layout.\n"
        "</system_notice>"
    ).mark_as_temp(),
]
```

该提示的稳定语义为：

- 使用 `<system_notice>` XML 包裹，说明输入是「被转换为单张图片的 GIF 动画」。
- 明确声明帧按阅读顺序排列。
- 指示模型把该图当作动画处理。
- 指示模型不要提及转换过程或帧布局。
- 通过 `.mark_as_temp()` 标记为临时内容，仅参与本轮请求，不写入持久化对话历史。

## 与上下文注入分层的关系

该提示的注入路径与 `@filter.on_llm_request()` 中推荐的 `extra_user_content_parts` 动态内容注入策略一致：追加在用户消息之后，不修改 `system_prompt`，因此不会破坏模型服务端的提示词缓存。

区别在于：本次由核心 Pipeline 自动注入，而不是由插件 Hook 注入；且使用 `.mark_as_temp()` 使提示不会持久化到 `conversation_manager` 的 history 中。

数据流为：

1. Platform Adapter 将平台原始消息转换为 `AstrBotMessage`，其中包含 `Image` 组件。
2. Pipeline 的 image_input 子阶段对图片引用执行 `prepare_model_image(...)`。
3. 动图被转换为 JPEG 拼图，并返回 `is_montage=True`。
4. `prepare_request_images(...)` 汇总所有图片后判断是否存在拼图。
5. 若存在，则向 `ProviderRequest.extra_user_content_parts` 追加临时 `TextPart` 系统提示。
6. ProviderRequest 进入 Chat Provider，由模型看到「动画拼图」提示与拼图图片。
7. 该提示不进入持久化历史，下一次请求会根据该轮输入重新判断。

## 插件开发契约

- 插件不应直接依赖 `prepare_model_image` 的新返回类型。该函数的返回类型变更属于核心内部工具契约变化；插件若确实调用该函数，需要按 `tuple[str, bool] | None` 解包。
- 插件不应自行追加同一拼图提示。核心会在检测到动画拼图时统一注入，重复注入会污染 `extra_user_content_parts` 并可能破坏模型对帧布局的解读。
- 插件依赖 `event.message_obj.message` 中 `Image` 组件时，其类型和构造方式没有变化。动画拼图只发生在请求构造阶段，不会改变事件模型。
- 当 `provider_settings.image_compress_enabled == False` 时，核心不会走 `prepare_model_image(...)` 分支（走 `MediaResolver` 原始路径），因此不会产生拼图提示。AI 助手在分析「为什么模型看不到动画拼图提示」时，应先确认该配置项。

## 变更影响分析

- **多帧输入不再被静默拼接**：模型现在知道输入是动画帧拼图，可按阅读顺序理解帧序列，避免把拼图读成单张静物。AI 开发者在分析模型「看图」行为时，应把拼图语义视为已知上下文的一部分。
- **`prepare_model_image` 返回值从 `str | None` 变为 `tuple[str, bool] | None`**：这是核心内部工具契约的非向后兼容变更。任何直接调用该函数的代码（包括插件或测试）必须解包为 `(path, is_montage)`。AI 助手在生成调用代码时不应假设返回裸路径字符串。
- **缓存命中同样报告拼图**：`is_montage` 由输入图片的帧数决定，与是否命中转换缓存无关。因此即使第二次请求直接复用缓存，`has_montage` 仍为真，提示仍会被追加。这避免了「第一次有提示、第二次丢失提示」的时序不一致。
- **提示是临时内容，不进入历史**：`TextPart.mark_as_temp()` 表明该提示不会持久化到对话历史。AI 助手在读取历史时应预期看不到该提示，但在运行时的 `ProviderRequest.extra_user_content_parts` 中可以看到。
- **提示条数按请求而非按图片计**：无论本轮请求包含几张动画拼图，`has_montage` 是布尔累加，因此最多追加一条提示。AI 助手在检查请求内容时不应期待「每张动图对应一条提示」。
- **无动图时不追加提示**：纯静态图请求不会追加该 `TextPart`，因此模型不会收到无意义的动画说明。这保证了静态图场景的上下文与环境完全一致。
- **与 `image_compress_enabled` 的交互**：只有在启用图片压缩流程时才会调用 `prepare_model_image(...)`，因此该提示只在压缩路径下出现。关闭图片压缩时，动图会以原始引用进入请求，不会出现系统提示。
- **不新增 Message Chain 组件或装饰器**：该变更没有引入新的 `Image` 子类型、新的 hook、新的 Provider 方法或新的 Adapter 接口。它仅扩展了图片准备阶段的返回值语义，并在 Pipeline 层追加了临时 `TextPart`。
- **模型缓存与成本不受影响**：该提示通过 `extra_user_content_parts` 注入，不修改 `system_prompt`，因此不触发提示词缓存失效；每轮请求独立计算是否追加，符合动态上下文注入的最佳实践。
- **排查动图识别问题时的顺序建议**：若模型对 GIF 表现出「看到一张图」的错误理解，应依次确认：`provider_settings.image_compress_enabled` 是否为真、`prepare_model_image` 返回的 `is_montage` 是否为真、`ProviderRequest.extra_user_content_parts` 中是否存在 `<system_notice>` 提示。
