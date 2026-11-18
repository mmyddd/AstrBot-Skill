---
title: SKILL.md Frontmatter Description 解析
type: improvement
status: stable
last_updated: 2026-06-24
related_base: agent/register skill.md
---

## 概述

AstrBot 在读取 Skill 的 `SKILL.md` 元数据时，已支持使用 YAML 语义解析 frontmatter 中的 `description` 字段。该字段不再只能是单行 `key: value` 形式，也可以使用 YAML 的多行字符串语法。

Skill 作者可以在 `SKILL.md` 顶部 frontmatter 中编写更长、更结构化的描述，供 Skill Manager 和相关 Agent 能力加载逻辑提取为 Skill 描述。

示例：

```yaml
---
name: humanizer-zh
description: |
  去除文本中的 AI 生成痕迹。
  适用于编辑或审阅文本，使其听起来更自然。
---
```

解析后的 `description` 为：

```text
去除文本中的 AI 生成痕迹。
适用于编辑或审阅文本，使其听起来更自然。
```

也支持 YAML folded block：

```yaml
---
name: humanizer-zh
description: >
  去除文本中的 AI 生成痕迹。
  适用于编辑或审阅文本，使其听起来更自然。
---
```

解析后的 `description` 为：

```text
去除文本中的 AI 生成痕迹。 适用于编辑或审阅文本，使其听起来更自然。
```

## 解析契约

Skill 元数据描述解析遵循以下稳定行为：

1. 只解析 `SKILL.md` 顶部由 `---` 包围的 YAML frontmatter。
2. 使用 YAML 解析结果读取 `description` 字段。
3. 当 `description` 是字符串时，返回 `description.strip()` 的结果。
4. 当 frontmatter 不存在、YAML 解析失败、解析结果不是对象，或 `description` 不是字符串时，返回空字符串。

这意味着 Skill 作者可以使用标准 YAML 字符串能力表达描述，包括：

- 单行字符串：`description: quoted value`
- literal block：`description: |`
- folded block：`description: >`

## 与 Agent / Skills 数据流的关系

插件可以通过 `skills/` 目录随插件提供 Skill。AstrBot 加载合法 Skill 后，会读取其中的 `SKILL.md`。`description` 是 Skill 元数据的一部分，会参与后续 Skill 管理和 Agent 能力描述构建。

此次变更将 `description` 的解析从简单的按行查找升级为 YAML frontmatter 解析，降低了 `SKILL.md` 文档表达形式与 AstrBot 解析器之间的耦合。Skill 作者不再需要为了兼容解析器而把长描述压缩成单行文本，可以直接使用 YAML 的多行语法保持可读性。

## 推荐写法

对于需要给 Agent 或开发者 AI 助手消费的 Skill，建议使用清晰、任务导向的描述：

```yaml
---
name: report-writer
description: |
  将输入材料整理为结构化报告。
  适合总结会议记录、技术调研和项目复盘。
---
```

描述应说明 Skill 的适用任务，而不是只写抽象人设。这样有助于 Agent 或其他 AI 消费者在检索、选择和组合 Skill 时理解其能力边界。

## 变更影响分析

- **多行 Skill 描述成为可用契约**：AI 开发者在生成 `SKILL.md` 时，可以安全使用 `description: |` 或 `description: >` 编写多行描述，不需要手动拼成单行。
- **YAML 语法错误会导致描述为空**：如果 frontmatter 不是合法 YAML，例如 `description: [broken`，解析结果会降级为空字符串。生成 Skill 文件时应确保 frontmatter 是合法 YAML。
- **非字符串 description 不会被接受**：如果 `description` 被写成数组、对象或其他非字符串类型，AstrBot 会返回空描述。Skill 描述应始终写为字符串。
- **literal 与 folded 语义不同**：`description: |` 会保留换行；`description: >` 会按 YAML folded block 规则折叠换行。AI 助手生成文档时应根据是否需要保留段落换行选择语法。
- **不改变 Skill 发现规则**：此变更只影响 `SKILL.md` frontmatter 中 `description` 的解析方式，不改变插件 `skills/` 目录发现机制，也不新增新的 Tool、Provider、Message Chain 或 Adapter API。
- **更适合 RAG 消费**：多行描述可以承载更完整的能力说明，减少 Skill 名称与简短单行描述不足造成的语义歧义。AI 助手在维护 Skill 文档时应优先写任务边界、输入类型和适用场景。
