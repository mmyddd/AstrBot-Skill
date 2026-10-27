---
title: Decision Provider 与 DecisionService
type: feature
status: stable
last_updated: 2026-10-27
related_base: agent/Invoke-llm.md
---

## 概述

AstrBot 新增系统一决策提供商类型 `ProviderType.DECISION`，并引入 `DecisionProvider` 抽象基类与 `DecisionService` 服务层。该能力用于把问题、选项或评分标准提交给决策 Provider，并接收概率、置信度或布尔判断结果。

核心数据流为：

```text
调用方 (DecisionService.ask / choose / yes_no)
  -> ProviderManager.get_using_provider_async(ProviderType.DECISION) 或 get_provider_by_id
      -> DecisionProvider.ask(questions, state)
          -> SystemOneDecisionProvider -> 远端 /systemone API
```

该 Provider 类型不是 Chat Provider，不应作为 `chat_provider_id` 传给 `context.llm_generate(...)` 或 `context.tool_loop_agent(...)`。它用于结构化决策调用，不是对话生成。

## 领域模型

实现位于 `astrbot/core/provider/entities.py`。

- `ProviderType` 新增 `DECISION`，值为 `decision`。
- 问题类型均为 `@dataclass(frozen=True, slots=True)`：
  - `DecisionChoiceQuestion(instructions: str, criteria: Mapping[str, str])`：至少 2 个选项，选项 ID 映射到描述。
  - `DecisionBooleanQuestion(instructions: str)`：布尔问题。
  - `DecisionScoreQuestion(instructions: str, criteria: list[str])`：2 到 10 个有序等级。
  - `DecisionQuestion: TypeAlias = Choice | Boolean | Score`。
- 上下文类型：`DecisionState: TypeAlias = str | list[Any] | dict[str, Any]`。
- 答案类型均为 frozen/slots：
  - `DecisionChoiceAnswer(choice: str, probability: float, probabilities: Mapping[str, float], confidence: float)`
  - `DecisionBooleanAnswer(probability: float)`
  - `DecisionScoreAnswer(score: float, legend: Mapping[str, str], probabilities: Mapping[str, float], confidence: float)`
  - `DecisionAnswer: TypeAlias = Choice | Boolean | Score Answer`。
- `DecisionError(RuntimeError)` 覆盖提供商不可用、请求失败、契约违规三类场景。

## DecisionService 服务层

新增文件 `astrbot/core/provider/decision.py`。`DecisionService(provider_manager: ProviderManager)` 把问题、提供商选择与契约校验从调用方解耦。

| 方法 | 签名要点 |
|---|---|
| `ask` | `(questions: Mapping[str, DecisionQuestion], state: DecisionState | None = None, *, provider_id: str | None = None) -> Mapping[str, DecisionAnswer]`。`provider_id` 缺省时走 `get_using_provider_async(ProviderType.DECISION)`；解析出的实例不是 `DecisionProvider` 时抛出 `DecisionError`。 |
| `choose` | `(question, criteria: Mapping[str, str] | Sequence[str], *, state=None, provider_id=None) -> DecisionChoiceAnswer`。criteria 规范化为 `o1..oN` 映射；要求至少 2 个且唯一；返回后校验 `choice` 与 `probabilities` 的键集，并映射回原始选项值。 |
| `yes_no` | `(question, *, state=None, threshold: float = 0.5, provider_id=None, default: bool | None = None) -> bool`。校验 question 非空、threshold 在 [0,1]；`probability >= threshold` 判真；遇到 `DecisionError` 时若提供 `default` 则记录 warning 并降级返回，否则重抛。 |

## DecisionProvider 基类与导出

- `astrbot/core/provider/provider.py` 新增 `DecisionProvider` 抽象基类，契约方法为 `ask(questions, state)`。
- `astrbot/core/provider/__init__.py` 的 `__all__` 增加 `DecisionProvider`。
- `astrbot/api/provider/__init__.py` 对外 API 暴露全部 `Decision*` 实体、`DecisionError` 与 `DecisionProvider`。

## ProviderManager 集成

`astrbot/core/provider/manager.py` 新增决策 Provider 状态与解析链路：

- 新增 `decision_provider_insts: list[DecisionProvider]` 与 `curr_decision_provider_inst: DecisionProvider | None`。
- `_resolve_using_provider`：`DECISION` 优先返回 `curr_decision_provider_inst`，否则回退 `decision_provider_insts[0]`。
- `initialize`：读取全局偏好 `curr_provider_decision`（`sp.get_async`，scope=global），从 `inst_map` 反解；失败时回退首个实例。
- `set_provider`：`DECISION` 分支写入 `curr_provider_decision` 并触发 `_notify_provider_changed`。
- `load_provider`：新增 `case ProviderType.DECISION`，强制校验 `issubclass(cls_type, DecisionProvider)`，实例化并调用 `initialize()`，入列并设置默认实例。
- `dynamic_import_provider`：新增 `system_one_decision` 分支，惰性导入 `sources.system_one_decision_source.SystemOneDecisionProvider`。
- `reload` 与 `terminate_provider`：同步维护 `decision_provider_insts` 与 `curr_decision_provider_inst`。

## 配置模板

`astrbot/core/config/default.py` 新增 4 个 `system_one_decision` 提供商模板：

| 模板 | id | provider | api_base | model |
|---|---|---|---|---|
| TypeSafe System One | `typesafe_decision` | typesafe | `https://api.typesafe.ai/v1/systemone` | `jev-latest` |
| OpenRouter System One | `openrouter_decision` | openrouter | `https://openrouter.ai/api/v1/systemone` | `jev-latest` |
| Vercel System One | `vercel_decision` | vercel | `https://ai-gateway.vercel.sh/typesafe/v1/systemone` | `typesafe-ai/jev` |
| Custom System One | `custom_decision` | custom | 空，需自填 | 空，需自填 |

公共字段包括 `type: system_one_decision`、`provider_type: decision`、`key: 空`、`timeout: 2`、`total_timeout: 5`、`max_concurrency: 4`、`proxy: 空`、`custom_headers: {}`。

新增配置项 Schema 包括：

- `total_timeout: float`：单次决策请求总超时，包含并发排队、重试与退避，默认 5 秒。
- `max_concurrency: int`：该提供商决策请求并发上限，默认 4。

## 变更影响分析

- **新增独立 Provider 类型**：`ProviderType.DECISION` 是决策调用入口，不是对话生成入口。AI 开发者在生成插件代码时，不应把 Decision Provider ID 传给 `context.llm_generate(...)` 或 `context.tool_loop_agent(...)` 的 `chat_provider_id`。
- **决策 API 具备契约校验**：`DecisionService.choose` 会校验 criteria 数量与唯一性，并校验返回 `choice` 与 `probabilities` 键集；非法响应会抛出 `DecisionError`，而不是静默返回。
- **yes_no 支持失败降级**：调用方可传入 `default` 作为 fail-open 或 fail-close 回退。未提供 `default` 时 `DecisionError` 会重抛，插件应显式处理异常。
- **决策结果不是确定性业务判断**：答案包含 probability、probabilities 或 confidence。AI 开发者在生成代码时，不应把决策结果当作严格规则引擎输出，应结合阈值、默认值或业务校验使用。
- **DecisionState 有类型边界**：状态仅支持 `str`、`list[Any]`、`dict[str, Any]`。不要假设可以传入任意对象。
- **超时与并发受配置约束**：`timeout`、`total_timeout`、`max_concurrency` 共同约束决策请求延迟与并发。默认超时较短，长耗时决策逻辑应依赖配置调整，而不是硬编码等待。
- **ProviderManager 生命周期扩展**：Decision Provider 与 Chat、STT、TTS 等 Provider 一样参与加载、切换、重载与终止。AI 助手理解 Provider 管理链路时，应把 Decision Provider 纳入同一套 `load_provider`、`set_provider`、`terminate_provider` 生命周期。
- **不改变 Message Chain 与 Adapter 契约**：本变更没有新增 Message Chain 组件、事件类型或 Platform Adapter 接口。插件仍通过既有 Provider 抽象与 Context 能力使用决策服务，不应直接调用远端 `/systemone` API。
