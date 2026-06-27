---
module: kb-templates
source: auto-generated
confidence: 1.0
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - templates/
---

# kb-templates — 模块概述

## 职责

本模块提供了 DevLoop 各阶段（逆向分析、需求摄入、验证等）使用的模板格式定义。
所有模板采用 Handlebars 风格的 `{{ }}` 变量占位符语法，由下游消费方（LLM、编排 Skill）读取模板后根据上下文填充变量生成具体文档。

## 模块类型

config

## 关键指标

- 文件数: 11
- 主要语言: Markdown / YAML
- 估计 Token 数: ~2200

## 模板分类

11 个模板按用途分为四个类别：

### 1. KB 条目模板（5 个）

用于阶段一（逆向分析）Step 2 生成 KB 知识库条目：

| 文件名 | 用途 |
|--------|------|
| `kb-module-overview.md` | 模块概述 — 职责、核心功能、对外接口、依赖关系 |
| `kb-module-api.md` | API 文档 — 导出符号、内部 API、参数/返回值 |
| `kb-module-data-model.md` | 数据模型 — 实体定义、字段、约束、关联、状态转换 |
| `kb-module-business-logic.md` | 业务逻辑 — 核心流程、业务规则、副作用 |
| `kb-module-dependencies.md` | 依赖关系 — 上下游依赖、耦合度、依赖图 |

### 2. 需求文档模板（2 个）

用于阶段二（需求摄入）需求录入和细化：

| 文件名 | 用途 |
|--------|------|
| `requirement-input.md` | 简易需求输入 — 标题、描述、验收标准 |
| `requirement-doc.md` | 完整需求文档 — 含 frontmatter、推断来源、API/数据模型变更、影响分析、实现跟踪 |

### 3. 分析/验证模板（3 个）

用于阶段二和阶段四的分析和验证：

| 文件名 | 用途 |
|--------|------|
| `impact-analysis.md` | 影响分析 — 涉及模块、API/数据模型变更、风险评估 |
| `nfr-checklist.md` | 非功能需求强制评估清单 — 7 个维度的评估框架和触发条件 |
| `verification-checklist.md` | 验证检查清单 — 自动化验证、验收标准、NFR 验证、代码审查、门禁 |

### 4. 清单模板（1 个）

| 文件名 | 用途 |
|--------|------|
| `kb-manifest.yaml` | KB 模块清单 YAML — 所有已分析模块的索引汇总 |

## 核心功能

- **结构化文档生成**: 提供统一格式的模板，确保所有产出物风格一致
- **变量占位符**: 使用 `{{ }}` 语法标记需要填充的变量，通过 Handlebars 风格迭代结构 (`{{#each}}`)
- **条件渲染**: 部分模板支持条件判断（如 `requirement-doc.md` 的 `{{#if status == "baseline"}}`），按数据状态渲染不同内容
- **YAML frontmatter**: 每个模板包含标准化的 frontmatter 元数据字段

## 模板文件命名规范

- `kb-module-*.md` — KB 条目模板（供 `devloop-reverse` Step 2 使用）
- `requirement-*.md` — 需求文档模板（供 `devloop-skills` 需求摄入使用）
- `impact-analysis.md` — 影响分析（供阶段二 Step 3 使用）
- `nfr-checklist.md` — NFR 评估指南（供阶段二 Step 4 使用）
- `verification-checklist.md` — 验证检查清单（供阶段四使用）
- `kb-manifest.yaml` — 模块清单汇总

## 对外接口

- `templates/kb-module-overview.md` — 提供模块概述格式
- `templates/kb-module-api.md` — 提供 API 文档格式
- `templates/kb-module-data-model.md` — 提供数据模型格式
- `templates/kb-module-business-logic.md` — 提供业务逻辑格式
- `templates/kb-module-dependencies.md` — 提供依赖关系格式
- `templates/requirement-input.md` — 提供需求输入格式
- `templates/requirement-doc.md` — 提供完整需求文档格式
- `templates/impact-analysis.md` — 提供影响分析格式
- `templates/nfr-checklist.md` — 提供 NFR 评估框架
- `templates/verification-checklist.md` — 提供验证检查清单格式
- `templates/kb-manifest.yaml` — 提供模块清单索引格式

## 依赖关系

- **devloop-skills**: 编排 Skill 读取模板生成各阶段产出物
- **devloop-reverse**: Step 2 使用 kb-module-* 模板生成 KB 条目

## 被依赖关系

无主动依赖关系。模板是纯数据定义，不依赖其他模块运行；由外部模块按需读取。

## 备注

- 所有模板均为纯格式定义，不含执行逻辑
- 变量类型和用途在模板底部的 `<!-- Variables: -->` HTML 注释中声明
- 部分变量带有条件渲染逻辑（`{{#if}}`, `{{#each}}`），由渲染引擎（如 LLM 或 Handlebars）解释
