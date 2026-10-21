---
title: 备份包含定时任务 (CronJob in Backups)
type: improvement
status: stable
last_updated: 2026-10-21
related_base: kv_storage.md
---

## 概述

AstrBot 的全量备份（导出 / 导入）现在包含定时任务表 `cron_jobs`。实现方式是将 `CronJob` 模型注册进备份模块的表清单 `MAIN_DB_MODELS`，从而让备份流水线以与其他主库表（`platform_stats`、`conversations`、`personas`、`persona_folders`、`preferences` 等）一致的方式处理定时任务数据。

该变更位于数据库备份层，不修改 Message Chain、消息组件、Platform Adapter、Provider 接口或插件装饰器。它改变的是“备份档案中包含哪些持久化数据”这一对外契约：带定时任务的部署在迁移、恢复或复制环境后，不再丢失已持久化的调度任务。

## 核心机制

备份模块通过一张集中式映射声明需要参与备份的主库表：

```python
MAIN_DB_MODELS: dict[str, type[SQLModel]] = {
    "platform_stats": PlatformStat,
    "conversations": ConversationV2,
    "cron_jobs": CronJob,
    "personas": Persona,
    "persona_folders": PersonaFolder,
    "preferences": Preference,
    ...
}
```

- 映射的 key（`"cron_jobs"`）是备份档案中该表对应的逻辑名称。
- 映射的 value（`CronJob`）是 `astrbot.core.db.po` 导出的 SQLModel 类型，供导出器与导入器统一做行级序列化和反序列化。

因此该表被纳入与既有主库表完全相同的导出/导入路径，而不是新增一套专用备份逻辑。`CronJob` 从 `astrbot.core.db.po` 与 `ConversationV2`、`Persona`、`PersonaFolder` 等模型一并导入到 `astrbot/core/backup/constants.py`，成为备份模型清单的一等成员。

## 数据流

1. 任务创建：调用方（核心或插件）在数据库中创建定时任务记录，例如测试中的 `create_cron_job(name=..., job_type="active_agent", cron_expression=..., timezone=..., payload=...)`。
2. 备份导出：备份导出器读取 `MAIN_DB_MODELS`，按表导出主库数据，`cron_jobs` 表随其他表一起写入备份档案。测试入口为 `AstrBotExporter(main_db=..., config_path=...).export_all(output_dir=...)`，返回 zip 备份文件路径。
3. 备份导入：备份导入器按同一份 `MAIN_DB_MODELS` 清单还原表数据，入口为 `AstrBotImporter(main_db=...).import_all(zip_path)`，返回带 `success` / `errors` 的结果对象。
4. 结果校验：目标库通过 `list_cron_jobs()` 读取任务列表，round-trip 测试断言恢复出的任务数量为 1，且 `restored[0].model_dump() == job.model_dump()`，即字段值与导出前完全一致。

该数据流说明备份契约是“表级、字段完整”的：`CronJob` 的 `name`、`job_type`、`cron_expression`、`timezone`、`payload` 等模型字段都会随备份往返，不依赖任务是否在运行期被重新注册。

## 与插件开发的关系

定时任务的注册与维护仍通过既有的 `cron_manager` 接口完成（见 `agent/cron.md`：`add_basic_job` / `add_active_job` / `list_jobs` / `delete_job` / `update_job`）。本次变更不新增任何 cron 相关方法或参数。

对插件作者可推导的稳定行为是：只要任务记录已写入 `cron_jobs` 表（例如 `add_basic_job` 传入 `persistent=True` 时依赖数据库持久化的任务，或 `add_active_job` 创建的主动任务），该任务就会随 AstrBot 全量备份进入备份档案，并在导入后被完整还原。插件不需要为“任务在备份后丢失”额外编写重建逻辑，也不应通过备份档案自行拼装任务表。

插件作者在以下场景应把备份视为任务数据的持久化边界之一：

- 环境迁移或灾备恢复后依赖任务继续调度。
- 测试中用备份档案复制一套任务配置。
- 排查“任务消失”问题时区分任务是被删除、未被持久化，还是未包含在导入的备份档案中。

## 变更影响分析

- **备份契约扩展**：备份档案现在包含 `cron_jobs` 表。AI 开发者在描述 AstrBot 备份内容时，应把定时任务列入与 conversation、persona、preference 同级的持久化数据，而不是遗漏。
- **持久化任务跨环境可迁移**：依赖数据库持久化的调度任务不再需要在新环境中手工重建。插件若在 `terminate()` 中清理任务，或把任务仅保存在内存中，则不属于该备份契约覆盖范围，仍需自行处理恢复逻辑。
- **不新增对外 API**：本次没有新增装饰器、`Context` 方法、Provider 接口、Message Chain 组件或 Adapter 接口。插件仍通过 `cron_manager` 管理任务，备份行为由核心备份层负责。
- **表级映射是唯一开关**：是否需要参与备份由 `MAIN_DB_MODELS` 决定。AI 助手在分析“某类数据为什么没有进入备份”时，应先检查该映射是否包含对应表，而不是先去排查导出代码分支。
- **字段完整性而非语义重建**：导入结果按模型字段逐一还原（测试用 `model_dump()` 全等断言）。这意味着任务的时间表达式、时区、payload 等原始配置被保留；但任务在恢复后何时被实际触发，仍由调度器按恢复出的表达式重新计算，备份不保存运行时触发状态。
- **排障边界**：若导入后 `list_cron_jobs()` 为空，可能原因包括备份档案由旧版本生成（旧版本未包含 `cron_jobs` 表）、任务从未写入该表（未持久化），或导入过程 `errors` 中出现对应表错误。
- **不改变任务身份语义**：备份往返的是任务记录本身，插件不应假设导入后可获得与原环境相同的任务 ID 之外的额外运行期状态；需要引用任务时仍应通过 `list_jobs()` 查询结果获取实际 `CronJob`。