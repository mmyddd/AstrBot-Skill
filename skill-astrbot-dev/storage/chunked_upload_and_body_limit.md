---
title: Dashboard 分片上传与请求体限制
category: storage
type: feature
status: stable
last_updated: 2026-09-18
related_base: file_storage.md
---

## 概述

AstrBot Dashboard 引入了统一的上传写入路径、可复用的分片上传会话服务，以及请求体大小限流中间件。该变更影响所有通过 Dashboard HTTP 接口上传文件的场景：备份导入（Backup）、插件配置/技能文件上传、以及聊天附件（Chat Attachments）上传。

核心目标是限制上传过程的内存占用，并为大文件提供可恢复的分片上传能力。插件作者接入 Web API 或处理上传时，应理解以下新增契约。

## 统一流式保存：`save_upload_stream`

新增模块 `astrbot/core/utils/upload.py`，提供单一流式保存实现：

- `UploadTooLargeError(ValueError)`：超限异常，携带 `max_bytes` 属性。
- `save_upload_stream(upload, dest, *, max_bytes: int | None = None) -> int`：以 `COPY_BLOCK_SIZE = 1 MiB` 的分块流式写入至唯一命名临时文件 `<name>.<uuid>.tmp`，成功后通过 `asyncio.to_thread(os.replace, ...)` **原子重命名** 至 `dest`；失败或取消时仅清理自身临时文件，不会破坏已存在的目标文件。超限时删除 partial 临时文件并抛出 `UploadTooLargeError`。返回写入字节数。

所有 `UploadFileAdapter.save()` 包装器（`astrbot/api/web.py`、`dashboard/api/multipart.py`、`dashboard/asgi_runtime.py`）的签名统一为：

```python
async def save(self, destination, *, max_bytes: int | None = None) -> int
```

内部全部委托 `save_upload_stream`，移除了各自重复的 1 MiB 分块拷贝逻辑。对插件 Web API 开发者而言，`UploadFileAdapter.save()` 现在是唯一的写入入口，且支持 `max_bytes` 上限参数。

## 请求体大小限流中间件

`dashboard/server.py` 新增 `dashboard_body_limit_middleware`，注册于 auth 中间件之后（最外层），在任何解析发生前拒绝超限请求，返回 `JSONResponse(error(message), status_code=...)`。

- 常量 `_MULTIPART_OVERHEAD_BYTES = 1 MiB`：为 multipart 边界/头部预留，避免恰好等于限值的文件被误判为 413。
- 常量 `_BODY_METHODS = {"POST", "PUT", "PATCH"}`。
- `_BODY_LIMIT_OVERRIDES` 按路径前缀覆盖默认 `MAX_CONTENT_LENGTH` 上限（更具体前缀在前），例如：
  - `.../backups/upload/chunk`、`.../files/upload/chunk` → `CHUNK_SIZE * 2`
  - `/api/v1/files`、`/api/chat/post_file`、`/api/v1/knowledge-bases/` → `MAX_UPLOAD_FILE_SIZE_BYTES + _MULTIPART_OVERHEAD_BYTES`
  - `/api/v1/plugins/config-files` → `MAX_FILE_BYTES + _MULTIPART_OVERHEAD_BYTES`
- `_check_body_limit(path, content_length, content_type, *, method, default_limit) -> tuple[int, str] | None`：
  - 仅作用于 `/api` 前缀。
  - 无 `Content-Length`：若方法属于 `_BODY_METHODS` 且 `parse_options_header` 判定 media type 为 `multipart/form-data`（容忍前导空格），返回 **411**；GET 等无体方法放行。
  - 有长度：按覆盖表前缀匹配取上限，否则使用 `self.app.config["MAX_CONTENT_LENGTH"]`，超限返回 **413**。

该中间件是两阶段上传判定的第一层，第二层由保存时的 `max_bytes` 兜底（应对无长度或伪造长度）。

## 分片上传会话服务

新增 `ChunkedUploadService`（`dashboard/services/chunked_upload_service.py`），负责分片会话生命周期、分片文件存储、流式拼接与过期清理。`BackupService` 将原先内联的 `upload_sessions` / `_cleanup_task` / 过期清理下沉至 `self.chunked_uploads = ChunkedUploadService(self.chunks_dir)`；`ensure_cleanup_task_started()` 与 `cleanup_upload_session(upload_id)` 变为委托调用；`math` / `shutil` / `time` 依赖及内联清理循环被移除。

所有分片会话接口新增 `owner: str` 参数（所有权隔离）：`upload_init` / `upload_chunk` / `upload_complete` / `upload_abort` / 新增 `upload_status` 均接受 `*, owner: str`。会话绑定到认证用户名（`auth.username` / `username`）；跨用户的 chunk/complete/abort/status 调用统一返回 "not found or expired"，防止 upload_id 被探测或跨用户消费。

`upload_status` 是新的查询接口：返回已接收分片索引、总数与过期信息，供客户端断点续传；查询为严格只读，不更新 `last_activity`，避免轮询延长会话与磁盘分片生命。

`BackupService` 新增常量：

- `MAX_BACKUP_TOTAL_BYTES = 8 GiB`：超限提示改用 FTP/SFTP 手工放入 backups 目录。
- `MAX_DIRECT_UPLOAD_BYTES = 128 MiB`：整文件上传上限，超出引导走分片。`upload_backup()` 改用 `file.save(zip_path, max_bytes=MAX_DIRECT_UPLOAD_BYTES)`，捕获 `UploadTooLargeError` 转 `BackupServiceError`。

备份分片路径下 `save_chunk()` 会先写入唯一临时文件、校验 `save()` 返回的字节数与预期 chunk 长度一致，再通过原子 rename 发布；失败或并发同索引重试不会截断并删除已接收分片。

## Dashboard 路由层

### `dashboard/api/backups.py`
- 所有路由从 `require_system_scope` / `require_dashboard_user` 注入的真实身份（`auth.username` / `username`）传入 `owner=`。
- `UploadFile` 参数统一经 `UploadFileAdapter(file)` 包装后再交给 service。
- **新增路由**：`POST /backups/upload/status`（system scope）与 legacy `POST /upload/status`，调用 `service.upload_status(_model_dict(payload), owner=...)`。

### `dashboard/api/files.py`（聊天文件分片上传）
- 新增辅助 `_run_chat_upload(operation)`，封装错误短路逻辑；`_upload_file` 重构为其调用者。
- **新增 5 个路由**（`require_file_scope`，均带 `owner=auth.username`）：
  - `POST /files/upload/init` — `service.upload_init(payload.model_dump(exclude_none=True), owner=...)`
  - `POST /files/upload/chunk` — 表单参数 `upload_id` / `chunk_index` / `chunk: UploadFile`（经 `UploadFileAdapter`）
  - `POST /files/upload/complete`
  - `POST /files/upload/abort`
  - `POST /files/upload/status`

### `dashboard/schemas.py`
新增 `ChatUploadInitRequest(OpenModel)`（`filename` / `total_size` / `content_type`）与 `ChatUploadSessionRequest(OpenModel)`（`upload_id`）。

`ChatService` 复用通用 `ChunkedUploadService`，并以新的 meta 透传方式把客户端 `content_type` 携带过会话；完成时在 chunk 目录之外合并，再经基于移动的 `LocalUploadFile` 适配器交给 `save_uploaded_file()`，保持 MIME 嗅探、类型分类与附件入库逻辑集中在一处。

## 数据流要点

1. **上传写入路径统一**：所有 `UploadFileAdapter.save` → `save_upload_stream` → temp 文件 + 原子 rename。
2. **两阶段上传判定**：中间件按 `Content-Length` 做早期 413/411 拒绝 → 保存时 `max_bytes` 二次兜底。
3. **分片会话所有权**：所有会话操作绑定 `owner`；`upload_status` 允许前端轮询恢复中断上传。
4. **聊天文件与备份共用** `ChunkedUploadService`，`CHUNK_SIZE = 1 MiB` 一致。

## 变更影响分析

- **插件上传代码不应再依赖文件名/后缀决定最终落盘路径**：经由 `UploadFileAdapter.save()` 写入时，字节先落唯一临时文件名，成功后原子 replace 至目标路径。插件若在保存过程中检查目标路径是否已存在，应预期它只会在成功 rename 后才出现。
- **`save()` 现在是 `max_bytes`-aware 契约**：插件 Web API 处理大文件上传时应显式传入 `max_bytes`，与 Dashboard 中间件的 413 形成双层防护。无长度或伪造长度的请求仍受保存期长度检查约束。
- **无 Content-Length 的 multipart 请求会被 411 拒绝**：中间件仅对 POST/PUT/PATCH 生效；GET 等无体方法不会触发该规则。插件若需要从浏览器上传大文件，请确保 HTTP 客户端携带 `Content-Length`，且 `Content-Type` 不含前导空格导致的解析错位。
- **分片上传接口只对认证用户有效**：所有分片接口都要求 `owner`，且 `upload_status` 的响应不包含文件名，仅暴露分片索引与进度信息，避免跨用户探测。插件开发者若在自定义端点复用 `ChunkedUploadService`，应保证调用方身份与 `owner` 严格对应。
- **聊天附件阈值**：前端对 ≥ 32 MiB 的附件切换到分片上传，小于该值的文件仍走单 POST 路径。插件消费聊天附件时无需感知此差异，最终附件仍通过 `save_uploaded_file()` 规范化入库。
- **备份整文件上传有 128 MiB 上限**：超过该值必须走分片；分片会话总大小上限为 `MAX_BACKUP_TOTAL_BYTES = 8 GiB`，更大会引导用户改用 FTP/SFTP 手工放入 backups 目录。
- **过期清理语义**：`cleanup_session()` 只在 `rmtree()` 成功后才注销会话，让 janitor 在瞬时文件系统错误时重试；会话在 idle 过期后仍可被 abort 清理，但 `get_session()` 会按错误信息所述真正拒绝已过期会话。
- **不新增 Message Chain / Provider / Adapter 接口**：该变更仅影响 Dashboard HTTP 层与上传存储层，不影响 `AstrMessageEvent`、`MessageChain`、`Provider` 调用或平台适配器契约。
