---
req_id: {{ req_id }}
req_title: {{ title }}
analysis_date: {{ analysis_date }}
analyzed_by: {{ analyzed_by }}
kb_contexts_used:
{{#each kb_contexts_used}}
  - {{ this }}
{{/each}}
---

# 影响分析: {{ title }}

## 涉及模块
{{#each affected_modules}}
### {{ this.name }}
- **影响程度**: {{ this.impact_level }}
- **变更描述**: {{ this.change_description }}
- **预计修改文件数**: {{ this.estimated_files }}
{{/each}}

## API 变更
{{#if has_api_changes}}
{{#each api_changes}}
### {{ this.method }} {{ this.path }}
- **变更类型**: {{ this.change_type }}
- **描述**: {{ this.description }}
- **兼容性**: {{ this.compatibility }}
{{/each}}
{{else}}
✅ 无 API 变更
{{/if}}

## 数据模型变更
{{#if has_data_model_changes}}
{{#each data_model_changes}}
### {{ this.entity }}
- **变更类型**: {{ this.change_type }}
- **字段变更**: {{ this.fields }}
- **迁移需求**: {{ this.migration_needed }}
{{/each}}
{{else}}
✅ 无数据模型变更
{{/if}}

## 风险评估
{{#each risks}}
- **风险**: {{ this.description }}
- **严重度**: {{ this.severity }}
- **缓解措施**: {{ this.mitigation }}
{{/each}}

## 非功能需求评估
<!-- 由阶段二 NFR 强制评估步骤填充 -->
{{ nfr_assessment }}

---
<!-- Variables:
  req_id / title — the requirement being analyzed
  analysis_date — ISO 8601 timestamp
  analyzed_by — "AI" | "human"
  kb_contexts_used — KB entries loaded for this analysis
  affected_modules — list of {name, impact_level, change_description, estimated_files}
  has_api_changes — boolean
  api_changes — list of {method, path, change_type, description, compatibility}
  has_data_model_changes — boolean
  data_model_changes — list of {entity, change_type, fields, migration_needed}
  risks — list of {description, severity, mitigation}
  nfr_assessment — non-functional requirements assessment (populated by Stage 2 NFR step)
-->
