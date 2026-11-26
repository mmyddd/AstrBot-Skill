---
title: Whisper API STT Opus 转 WAV 行为
type: improvement
status: stable
last_updated: 2026-06-24
related_base: agent/Invoke-llm.md
---

## 概述

`ProviderOpenAIWhisperAPI` 在执行 STT 转写时，现在会在调用 Whisper API 前自动处理 `.opus` 音频文件。

核心入口仍是现有 STT Provider 方法：

```python
async def get_text(self, audio_url: str) -> str
```

当 `audio_url` 指向本地 `.opus` 文件时，Provider 会先将该文件转换为临时 `.wav` 文件，再把转换后的 WAV 文件提交给 OpenAI Whisper transcription 接口。该变更主要修复 Feishu / Lark 等平台可能产生 Opus 语音文件时，Whisper API STT 无法直接稳定消费原始文件的问题。

插件作者和平台适配器作者不需要新增 Message Chain 组件，也不需要绕过 AstrBot 的 Provider 抽象手动调用 Whisper API。只要音频文件路径最终进入 `ProviderOpenAIWhisperAPI.get_text(...)`，`.opus` 文件会在 Provider 层被归一化为 `.wav` 后再转写。

## 核心行为

实现位置：

```python
astrbot/core/provider/sources/whisper_api_source.py
```

`get_text(...)` 会先确认传入的本地音频文件存在。随后对文件路径执行小写化判断：

```python
lower_audio_url = audio_url.lower()
```

当文件名以 `.opus` 结尾时，执行以下流程：

1. 通过 `get_astrbot_temp_path()` 获取 AstrBot 临时目录。
2. 在临时目录中生成形如 `whisper_api_<uuid>.wav` 的输出路径。
3. 调用 `convert_audio_to_wav(audio_url, output_path)` 将 Opus 文件转换为 WAV。
4. 将 `audio_url` 更新为转换后的 `.wav` 文件路径。
5. 后续转写流程使用转换后的 WAV 文件调用 Whisper transcription API。

该逻辑发生在旧有 `.amr`、`.silk` 和腾讯语音格式转换分支之前，因此 `.opus` 文件会优先走通用音频转 WAV 路径，而不是进入 AMR / SILK / Tencent 专用转换逻辑。

## 与 STT Provider 数据流的关系

该变更位于 Provider 层，而不是 Platform Adapter、Message Chain 或插件事件系统层。

典型数据流为：

1. Platform Adapter 或上游消息处理逻辑获得语音文件路径。
2. STT Provider 的 `ProviderOpenAIWhisperAPI.get_text(audio_url)` 接收该路径。
3. 如果路径指向 `.opus` 文件，Provider 在本地临时目录中生成 WAV 文件。
4. Provider 使用转换后的 WAV 文件调用 `client.audio.transcriptions.create(...)`。
5. Whisper API 返回转写文本。
6. Provider 返回最终字符串结果给上层调用方。
7. 转换产生的临时 WAV 文件会在转写完成后被清理。

这保持了 AstrBot 的分层边界：平台适配器只负责把平台消息接入统一消息模型和语音处理链路；音频格式归一化由 STT Provider 在请求外部模型前完成；插件作者继续通过 Provider 抽象消费语音转文字能力。

## 对插件和适配器开发的影响

对于插件作者，现有调用方式不变。插件仍应优先依赖 AstrBot 的 STT Provider，而不是在业务逻辑中为 Feishu / Lark 的 Opus 文件编写专属转换分支。

对于平台适配器作者，如果平台语音消息最终落地为 `.opus` 本地文件，并交给 OpenAI Whisper API STT Provider 转写，则 Provider 会负责转换为 WAV。适配器仍需确保传入的是可访问的本地文件路径，因为 `get_text(...)` 会在处理前检查文件是否存在。

该变更不会新增独立的 `Opus` 消息组件，也不会改变 `Record`、`Message Chain` 或 `AstrMessageEvent` 的统一消息语义。Opus 到 WAV 的转换是 STT Provider 内部的输入兼容逻辑。

## 变更影响分析

- **Whisper API STT 对 Opus 输入更兼容**：`.opus` 文件会在 Provider 层自动转换为 `.wav` 后再提交转写，减少 Feishu / Lark 语音文件直接进入 Whisper API 时的格式兼容问题。
- **插件 API 没有变化**：本变更没有新增装饰器、Message Chain 组件、Context 方法或 Platform Adapter 接口。AI 开发者生成插件代码时仍应使用既有 Provider / STT 抽象，不应直接依赖 OpenAI 客户端。
- **转换只针对本地 `.opus` 路径**：`get_text(...)` 的该分支通过文件名后缀判断 `.opus`，并且在转换前要求 `audio_url` 对应的文件存在。调用方应保证传入的是可读取的本地文件路径。
- **后缀判断大小写不敏感**：Provider 会使用 `audio_url.lower()` 判断扩展名，因此 `.OPUS`、`.Opus` 等大小写形式也会进入 Opus 转换路径。
- **Opus 优先于 AMR / SILK 专用转换逻辑**：`.opus` 文件会先进入 `convert_audio_to_wav(...)` 分支，不会落入 `.amr`、`.silk` 或腾讯语音格式处理分支。AI 助手在分析音频转换路径时应按该顺序理解。
- **临时 WAV 是 Provider 内部中间产物**：转换输出路径位于 AstrBot 临时目录，文件名形如 `whisper_api_<uuid>.wav`。插件不应依赖该临时文件路径作为业务状态或长期存储。
- **转写请求使用 WAV 文件名语义**：测试验证转写请求中传入的文件参数名为 `audio.wav`，底层文件对象来自转换后的 `.wav` 文件。这说明外部 Whisper transcription 调用看到的是 WAV 输入，而不是原始 Opus 文件。
- **音频转换失败会阻断转写**：如果 `convert_audio_to_wav(...)` 无法成功生成 WAV，后续 Whisper API 调用无法获得有效输入。插件若需要用户可见的错误提示，应在调用 STT 能力的上层捕获异常并返回明确反馈。
