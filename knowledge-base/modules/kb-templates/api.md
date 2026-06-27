---
module: kb-templates
source: auto-generated
confidence: 1.0
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - templates/
---

# kb-templates — API 文档

## 概述

每个模板的"接口"即其变量清单，声明于模板底部的 `<!-- Variables: -->` HTML 注释中。
下方列出每个模板的所有变量名、类型和用途。标记为 **必需** 的变量应在填充时保证提供；
标记为 **可选** 的变量可根据上下文省略（对应模板段落将被渲染为空或跳过）。

---

## 1. kb-module-overview.md — 模块概述模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `module_name` | string | 模块名称 | **必需** |
| `module_path` | string | 文件系统路径 | **必需** |
| `module_type` | string (enum) | 模块类型：service \| gateway \| library \| utility \| ui \| data \| config \| other | **必需** |
| `confidence` | float (0.0-1.0) | 自动生成置信度 | **必需** |
| `drift_score` | float (0.0-1.0) | 与当前代码的漂移程度 | **必需** |
| `last_updated` | string (ISO 8601) | 最后更新时间戳 | **必需** |
| `generated_from` | string[] | 源文件列表 | **必需** |
| `responsibility` | string | 模块职责描述 | **必需** |
| `file_count` | integer | 模块文件数 | **必需** |
| `language` | string | 主要编程语言 | **必需** |
| `token_estimate` | integer | KB 条目估算 Token 数 | **必需** |
| `core_functions` | {name, summary}[] | 核心功能列表 | **必需** |
| `public_apis` | {signature, description}[] | 对外接口列表 | 可选 |
| `dependencies` | {module, relationship}[] | 依赖关系 | 可选 |
| `dependents` | {module, relationship}[] | 被依赖关系 | 可选 |
| `notes` | string | 附加备注 | 可选 |

---

## 2. kb-module-api.md — API 文档模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `module_name` | string | 模块名称 | **必需** |
| `confidence` | float (0.0-1.0) | 置信度 | **必需** |
| `drift_score` | float (0.0-1.0) | 漂移分数 | **必需** |
| `last_updated` | string (ISO 8601) | 更新时间戳 | **必需** |
| `language` | string | 主要语言（用于示例代码块标记） | 可选 |
| `exports` | {name, kind, signature, description, parameters[], return_type, return_description, throws, example}[] | 导出符号列表 | **必需** |
| `internal_apis` | {name, signature, purpose}[] | 内部 API 列表 | 可选 |

其中 `exports` 的子字段：

| 子字段 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `name` | string | 符号名称 | **必需** |
| `kind` | string | 类型（function / class / interface / type / constant） | **必需** |
| `signature` | string | 完整签名 | **必需** |
| `description` | string | 描述 | **必需** |
| `parameters` | {name, type, description}[] | 参数列表 | 可选 |
| `return_type` | string | 返回值类型 | 可选 |
| `return_description` | string | 返回值说明 | 可选 |
| `throws` | string | 异常说明 | 可选 |
| `example` | string | 使用示例 | 可选 |

---

## 3. kb-module-data-model.md — 数据模型模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `module_name` | string | 模块名称 | **必需** |
| `confidence` | float (0.0-1.0) | 置信度 | **必需** |
| `drift_score` | float (0.0-1.0) | 漂移分数 | **必需** |
| `last_updated` | string (ISO 8601) | 更新时间戳 | **必需** |
| `entities` | {name, entity_type, description, fields[], constraints[], relations[]}[] | 实体列表 | **必需** |
| `state_machines` | {entity, diagram}[] | 状态转换图 | 可选 |

其中 `entities` 的子字段：

| 子字段 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `name` | string | 实体名称 | **必需** |
| `entity_type` | string | 实体类型（class / interface / table / struct） | **必需** |
| `description` | string | 实体描述 | **必需** |
| `fields` | {name, type, required, description}[] | 字段列表 | 可选 |
| `constraints` | string[] | 约束列表 | 可选 |
| `relations` | {target, type, description}[] | 关联关系 | 可选 |

---

## 4. kb-module-business-logic.md — 业务逻辑模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `module_name` | string | 模块名称 | **必需** |
| `confidence` | float (0.0-1.0) | 置信度 | **必需** |
| `drift_score` | float (0.0-1.0) | 漂移分数 | **必需** |
| `last_updated` | string (ISO 8601) | 更新时间戳 | **必需** |
| `core_flows` | {name, trigger, preconditions, steps[], postconditions, exceptions[]}[] | 核心流程列表 | **必需** |
| `business_rules` | {name, rule}[] | 业务规则列表 | 可选 |
| `side_effects` | string[] | 副作用描述列表 | 可选 |

其中 `core_flows` 的子字段：

| 子字段 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `name` | string | 流程名称 | **必需** |
| `trigger` | string | 触发条件 | **必需** |
| `preconditions` | string | 前置条件 | 可选 |
| `steps` | {index, description}[] | 流程步骤 | **必需** |
| `postconditions` | string | 后置条件 | 可选 |
| `exceptions` | {condition, handling}[] | 异常处理 | 可选 |

---

## 5. kb-module-dependencies.md — 依赖关系模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `module_name` | string | 模块名称 | **必需** |
| `confidence` | float (0.0-1.0) | 置信度 | **必需** |
| `drift_score` | float (0.0-1.0) | 漂移分数 | **必需** |
| `last_updated` | string (ISO 8601) | 更新时间戳 | **必需** |
| `upstream` | {module, dependency_type, what, coupling, interface}[] | 上游依赖列表 | **必需** |
| `downstream` | {module, dependency_type, what, coupling}[] | 下游依赖列表 | **必需** |
| `dependency_graph` | string (ASCII) | 依赖图文本表示 | 可选 |
| `has_circular` | boolean | 是否有循环依赖 | **必需** |
| `circular_deps` | string[] | 循环依赖描述 | 当 `has_circular=true` 时必需 |

---

## 6. requirement-input.md — 需求输入模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `type` | string (enum) | 需求类型：feature \| bugfix \| enhancement \| refactor \| chore | **必需** |
| `priority` | string (enum) | 优先级：high \| medium \| low | **必需** |
| `source` | string (enum) | 来源：cli \| api \| ide \| webhook \| manual | **必需** |
| `related_modules` | string (comma-separated) | 关联模块 | 可选 |
| `created_date` | string (ISO 8601) | 创建时间 | **必需** |
| `title` | string | 需求标题 | **必需** |
| `background` | string | 背景说明 | 可选 |
| `description` | string | 详细描述 | **必需** |
| `acceptance_criteria` | string[] | 验收标准列表 | 可选 |
| `additional_info` | string | 附加信息（链接、截图等） | 可选 |

---

## 7. requirement-doc.md — 完整需求文档模板

### Frontmatter 变量

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `req_id` | string | 唯一需求 ID（如 REQ-001） | **必需** |
| `title` | string | 需求标题 | **必需** |
| `type` | string (enum) | feature \| bugfix \| enhancement \| refactor \| chore | **必需** |
| `priority` | string (enum) | high \| medium \| low | **必需** |
| `status` | string (enum) | baseline \| proposed \| designed \| planned \| building \| verifying \| completed \| blocked \| superseded | **必需** |
| `primary_module` | string | 主要影响模块 | **必需** |
| `source` | string (enum) | auto-inferred \| user-submitted \| external-import | **必需** |
| `usage_restriction` | string (enum) | reference_only \| none （baseline 时恒为 reference_only） | **必需** |
| `kb_contexts` | string[] | 相关 KB 文件路径数组 | 可选 |
| `confidence` | string (enum) | high \| medium \| low | **必需** |
| `created_date` | string (ISO 8601) | 创建时间 | **必需** |
| `updated_date` | string (ISO 8601) | 更新时间 | **必需** |

### 内容体变量

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `inferred_from` | string | 推断来源（代码文件引用，仅 baseline） | 当 `status=baseline` 时必需 |
| `background` | string | 需求背景 | 可选 |
| `description` | string | 功能描述 | **必需** |
| `affected_modules` | {name, change_description}[] | 涉及模块及变更描述 | 可选 |
| `api_changes` | {method, path, change_type, description}[] | API 变更详情 | 可选 |
| `data_model_changes` | {entity, change_type, fields}[] | 数据模型变更 | 可选 |
| `acceptance_criteria` | string[] | 验收标准 | **必需** |
| `nfr_assessment` | string | 非功能需求评估（由阶段二填充） | 后填充 |
| `impact_analysis` | string | 影响分析（由阶段二填充） | 后填充 |
| `design_status/design_date` | string | Design 阶段状态/时间 | 后填充 |
| `plan_status/plan_date` | string | Plan 阶段状态/时间 | 后填充 |
| `build_status/build_date` | string | Build 阶段状态/时间 | 后填充 |
| `verify_status/verify_date` | string | Verify 阶段状态/时间 | 后填充 |

---

## 8. impact-analysis.md — 影响分析模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `req_id` | string | 需求 ID | **必需** |
| `title` | string | 需求标题 | **必需** |
| `analysis_date` | string (ISO 8601) | 分析时间 | **必需** |
| `analyzed_by` | string (enum) | AI \| human | **必需** |
| `kb_contexts_used` | string[] | 使用的 KB 条目路径 | 可选 |
| `affected_modules` | {name, impact_level, change_description, estimated_files}[] | 涉及模块 | **必需** |
| `has_api_changes` | boolean | 是否有 API 变更 | **必需** |
| `api_changes` | {method, path, change_type, description, compatibility}[] | API 变更详情 | 当 `has_api_changes=true` 时必需 |
| `has_data_model_changes` | boolean | 是否有数据模型变更 | **必需** |
| `data_model_changes` | {entity, change_type, fields, migration_needed}[] | 数据模型变更 | 当 `has_data_model_changes=true` 时必需 |
| `risks` | {description, severity, mitigation}[] | 风险评估 | 可选 |
| `nfr_assessment` | string | NFR 评估结果（由阶段二 NFR 步骤填充） | 后填充 |

---

## 9. nfr-checklist.md — 非功能需求评估清单

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `assessment_timestamp` | string (ISO 8601) | 评估时间戳 | **必需** |
| 各维度 `triggered` | string | ✅ 触发 \| ❌ 未触发 | **必需** |
| 各维度 `trigger_reason` | string | 触发/未触发原因 | **必需** |
| 各维度 `assessment_content` | string | 评估内容和补充验收标准 | 当触及时必需 |

> 注意：nfr-checklist.md 更多是一个**评估框架**（含 7 个维度及其触发条件），而非普通模板。它定义了评估流程、触发规则和不适用声明规则。变量仅用于最终的评估产出格式。

---

## 10. verification-checklist.md — 验证检查清单模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `req_id` | string | 需求 ID | **必需** |
| `title` | string | 需求标题 | **必需** |
| `verification_date` | string (ISO 8601) | 验证时间 | **必需** |
| `verified_by` | string (enum) | AI \| human | **必需** |
| `acceptance_criteria` | string[] | 验收标准（从需求文档继承） | **必需** |
| `nfr_checks` | {dimension, check_item}[] | NFR 验证检查项 | 可选 |
| `fix_attempts` | integer | 修复次数 | **必需** |
| `g4_1_result` | string (enum) | G4.1 自动化验证门禁：PASS \| WARN \| FAIL | **必需** |
| `g4_1_note` | string | 备注 | 可选 |
| `g4_2_result` | string (enum) | G4.2 代码审查门禁：PASS \| WARN \| FAIL | **必需** |
| `g4_2_note` | string | 备注 | 可选 |
| `g4_3_result` | string (enum) | G4.3 修复循环门禁：PASS \| WARN \| FAIL | **必需** |
| `g4_3_note` | string | 备注 | 可选 |
| `g4_4_result` | string (enum) | G4.4 阶段四总门禁：PASS \| WARN \| FAIL | **必需** |
| `g4_4_note` | string | 备注 | 可选 |

---

## 11. kb-manifest.yaml — 模块清单模板

| 变量名 | 类型 | 用途 | 必需性 |
|--------|------|------|--------|
| `modules` | list | 所有模块的 YAML 条目（被注释掉的模板格式） | **必需** |
| `generated_at` | string (ISO 8601) | 生成时间 | **必需** |
| `generated_by` | string | 生成者标识 | **必需** |
| `total_modules` | integer | 模块总数 | **必需** |

每个模块条目（被注释掉的模板格式）包含：

| 字段 | 类型 | 用途 |
|------|------|------|
| `module_name` | string | 模块名称 |
| `module_path` | string | 路径 |
| `module_type` | string | 模块类型 |
| `keywords` | string | 关键词 |
| `entities` | string | 实体 |
| `apis` | string | API |
| `file_count` | integer | 文件数 |
| `total_tokens` | integer | 总 Token 估算 |
| `summary_tokens` | integer | 摘要 Token 估算 |
| `last_analyzed` | string | 最后分析时间 |
| `confidence` | string | 置信度 |

---

## 必填/可选总结原则

- **Frontmatter 字段**: 绝大多数是必填，因为它们构成文档的核心元数据
- **`confidence`、`drift_score`、`last_updated`**: 在 KB 条目模板中统一必填
- **列表型变量**: 如 `exports`、`entities`、`core_flows` 等为必填，但内部子字段中非核心字段（如 `example`、`throws`、`constraints`）为可选
- **`notes` 型变量**: 始终可选
- **后填充变量**: 如 `nfr_assessment`、`impact_analysis`、门禁结果等，由后续阶段自动填充
