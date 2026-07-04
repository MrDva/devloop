---
id: {{ req_id }}
title: {{ title }}
type: {{ type }}
priority: {{ priority }}
status: {{ status }}
module: {{ primary_module }}             # 旧模块名或新业务功能名（或 "devloop-framework"）
business_function: {{ business_function }}     # 🆕 v2: 关联的业务功能名（v1 中为空）
source: {{ source }}
usage_restriction: {{ usage_restriction }}
kb_context:
{{#each kb_contexts}}
  - {{ this }}
{{/each}}
confidence: {{ confidence }}
created: {{ created_date }}
updated: {{ updated_date }}
---

{{#if status == "baseline"}}
> ⚠️ **AUTO-INFERRED BASELINE** | confidence: {{ confidence }} | source: code reverse-engineering
>
> This requirement was inferred from existing code implementation, NOT from original design documents or specifications.
> It may not reflect original design intent.
>
> **Usage restriction (`usage_restriction: reference_only`)**: This baseline requirement serves as **documentation reference only**.
> It MUST NOT be used as a mandatory constraint when designing new features in Stage 3 (Build).
> If a new requirement conflicts with a baseline requirement, the baseline requirement gives way.
>
> **source**: `auto-inferred` — derived from code analysis, not from original specifications.
> **usage_restriction**: `reference_only` — for background understanding only. Do NOT enforce as design constraint.
{{/if}}

# {{ req_id }}: {{ title }}

## 推断来源
{{ inferred_from }}

## 背景
{{ background }}

## 功能描述
{{ description }}

## 涉及模块
{{#each affected_modules}}
- **{{ this.name }}**: {{ this.change_description }}
{{/each}}

## API 变更
{{#each api_changes}}
### {{ this.method }} {{ this.path }}
- 变更类型: {{ this.change_type }}
- 描述: {{ this.description }}
{{/each}}

## 数据模型变更
{{#each data_model_changes}}
### {{ this.entity }}
- 变更类型: {{ this.change_type }}
- 字段: {{ this.fields }}
{{/each}}

## 验收标准
{{#each acceptance_criteria}}
- [ ] {{ this }}
{{/each}}

## 非功能需求评估
<!-- 由阶段二 NFR 强制评估步骤自动填充 -->
{{ nfr_assessment }}

## 影响分析
<!-- 由阶段二自动填充 -->
{{ impact_analysis }}

## 实现跟踪
| 阶段 | 状态 | 完成时间 |
|------|------|---------|
| Design | {{ design_status }} | {{ design_date }} |
| Plan | {{ plan_status }} | {{ plan_date }} |
| Build | {{ build_status }} | {{ build_date }} |
| Verify | {{ verify_status }} | {{ verify_date }} |

---
<!-- Variables:
  req_id       — unique requirement ID (e.g., REQ-001)
  title        — requirement title
  type         — feature | bugfix | enhancement | refactor | chore
  priority     — high | medium | low
  status       — baseline | proposed | designed | planned | building | verifying | completed | blocked | superseded
  primary_module — main affected module name
  source       — auto-inferred | user-submitted | external-import
  usage_restriction — reference_only | none (for baseline: always reference_only)
  kb_contexts  — array of KB file paths relevant to this requirement
  confidence   — high | medium | low (for baseline: must include confidence level)
  inferred_from — code files this was inferred from (baseline only)
  background   — why this requirement exists
  description  — detailed functional description
  affected_modules — list of {name, change_description}
  api_changes  — list of {method, path, change_type, description}
  data_model_changes — list of {entity, change_type, fields}
  acceptance_criteria — list of verifiable criteria
  nfr_assessment — non-functional requirements assessment (populated by Stage 2)
  impact_analysis — impact analysis content (populated by Stage 2)
  design_status / design_date — Design phase tracking
  plan_status / plan_date — Plan phase tracking
  build_status / build_date — Build phase tracking
  verify_status / verify_date — Verify phase tracking
  created_date / updated_date — ISO 8601 timestamps
-->
