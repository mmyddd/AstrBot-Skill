---
title: Local Computer 文件与 Shell 输出编码回退
type: improvement
status: stable
last_updated: 2026-06-24
related_base: agent/sandbox.md
---

## 概述

AstrBot 的 Local Computer 运行时在读取本地文件和解码本地 Shell 输出时，新增了统一的字节解码回退逻辑。该变更主要影响基于 Local runtime 的 Agent / Tool 执行链路，尤其是在 Windows 环境中读取 UTF-8、带 BOM 的 UTF-8、GBK、GB18030 或系统区域编码文件时的稳定性。

核心变化集中在：

```python
astrbot/core/computer/booters/local.py
```

新增内部解码函数：

```python
def _decode_bytes_with_fallback(
    output: bytes | None,
    *,
    preferred_encoding: str | None = None,
) -> str
```

`LocalFileSystemComponent.read_file(...)` 现在不再直接用指定编码打开文本文件，而是先以二进制读取文件内容，再通过 `_decode_bytes_with_fallback(...)` 解码：

```python
async def read_file(self, path: str, encoding: str = "utf-8") -> dict[str, Any]
```

这意味着插件、Tool 或 Agent 在 Local Computer 运行时读取本地文本文件时，对 Windows 本地编码文件的兼容性更强；读取结果仍通过原有返回结构暴露：

```python
{"success": True, "content": content}
```

## 解码顺序

`_decode_bytes_with_fallback(...)` 的稳定行为如下：

1. 当输入 `output is None` 时，返回空字符串。
2. 优先尝试调用方传入的 `preferred_encoding`。
3. 继续尝试 `utf-8`。
4. 继续尝试 `utf-8-sig`。
5. 在 Windows 环境下，继续尝试：
   - `mbcs`
   - `cp936`
   - `gbk`
   - `gb18030`
   - `locale.getpreferredencoding(False)` 返回的系统首选编码
6. 在非 Windows 环境下，尝试系统首选编码。
7. 如果全部失败，使用：

```python
output.decode("utf-8", errors="replace")
```

作为最终降级路径。

该顺序明确保证：即使 Windows 系统区域编码是 `cp936` 或其他本地编码，UTF-8 文件仍会在 Windows 本地编码之前被优先正确解码。

## LocalFileSystemComponent.read_file 行为

`LocalFileSystemComponent.read_file(...)` 的方法签名保持不变：

```python
async def read_file(self, path: str, encoding: str = "utf-8") -> dict[str, Any]
```

变化发生在内部读取方式：

- 旧行为：使用 `open(abs_path, encoding=encoding)` 直接按指定编码读取文本。
- 新行为：使用 `open(abs_path, "rb")` 读取原始字节，再调用 `_decode_bytes_with_fallback(raw_content, preferred_encoding=encoding)`。

因此，`encoding` 参数现在更准确地表示“优先尝试的编码”，而不是唯一可接受的编码。若指定编码无法解码文件内容，Local runtime 会继续尝试其他常见编码。

这对 Skill 文件、Agent 工具读取的文本文件、Windows 本地生成的 GBK 文件等场景更友好。测试覆盖了两个关键行为：

- Windows 系统首选编码为 `cp936` 时，UTF-8 文件仍优先按 UTF-8 正确读取。
- Windows 系统首选编码为 `cp1252` 时，GBK 文件可通过 Windows 中文编码回退路径正确读取。

## LocalShellComponent 输出解码行为

Shell 输出解码也复用了同一套回退函数：

```python
def _decode_shell_output(output: bytes | None) -> str:
    return _decode_bytes_with_fallback(output, preferred_encoding="utf-8")
```

这表示 Local Shell 执行结果仍优先按 UTF-8 解码，但在 Windows 环境中，如果输出字节不是 UTF-8，也会继续尝试 `mbcs`、`cp936`、`gbk`、`gb18030` 和系统首选编码。

该行为改善了 Local Computer runtime 中 Shell 命令输出进入 Agent / Tool 结果链路时的可读性，减少中文 Windows 环境下出现乱码或解码失败的概率。

## 与 Agent / Sandbox 数据流的关系

该变更位于 Local Computer booter 层，而不是插件事件系统、Message Chain、Provider 或 Platform Adapter 层。

典型数据流为：

1. Agent 或 Tool 在 Local Computer runtime 中请求读取本地文件，或执行本地 Shell 命令。
2. `LocalFileSystemComponent.read_file(...)` 或 `LocalShellComponent` 获取原始 `bytes`。
3. `_decode_bytes_with_fallback(...)` 根据优先编码、UTF-8、UTF-8 BOM、Windows 本地编码和系统首选编码依次尝试解码。
4. 解码后的字符串作为文件内容或 Shell 输出返回给调用方。
5. 上层 Agent / Tool 继续把该字符串作为工具结果、上下文内容或后续处理输入使用。

因此，该变更降低了 Local Computer runtime 与操作系统区域编码之间的耦合。插件作者和 AI 助手不需要在每个 Tool 中手动判断 Windows 编码，也不需要为常见 UTF-8 / GBK 文件分别写多套读取逻辑。

## 变更影响分析

- **Windows 文本读取更稳定**：在 Local runtime 中读取 Skill 文件或普通文本文件时，UTF-8、UTF-8 BOM、GBK、GB18030 等常见编码更可能被自动正确解码。
- **`encoding` 参数语义变为“优先编码”**：`LocalFileSystemComponent.read_file(path, encoding="...")` 不再只按该编码读取；如果失败，会继续尝试 fallback 编码。AI 开发者不应把该参数理解为严格编码校验。
- **UTF-8 优先级明确高于 Windows 本地编码**：即使 Windows locale 返回 `cp936`，UTF-8 文件也会先按 UTF-8 解码。这避免中文 Windows 环境下把 UTF-8 文件误读为本地编码。
- **Shell 输出乱码概率降低**：本地 Shell 输出如果不是 UTF-8，在 Windows 上会继续尝试中文相关编码。Tool 作者在处理 Local Shell 输出时可以更依赖返回字符串本身，而不是自行对 bytes 做平台分支。
- **最终降级会替换非法字符**：当所有候选编码都失败时，系统使用 `utf-8` + `errors="replace"` 返回字符串。这保证调用链不会因为解码失败直接中断，但内容中可能出现替换字符。
- **不新增插件 API**：本变更没有新增装饰器、Message Chain 组件、Provider 方法或 Platform Adapter 接口。它改变的是 Local Computer runtime 内部把 bytes 转为 str 的稳定性。
- **不要在插件层重复实现编码探测**：如果插件只是通过 Local runtime 读取文件或获取 Shell 输出，应优先使用现有返回内容。只有在业务确实要求严格区分编码时，才需要在插件内部自行读取原始文件并实现更严格的校验逻辑。
- **阻塞行为没有变化**：文件读取仍通过 `asyncio.to_thread(...)` 包装同步文件 I/O。该变更改善解码兼容性，但不改变 AstrBot 插件应避免阻塞 Event Loop 的基本要求。