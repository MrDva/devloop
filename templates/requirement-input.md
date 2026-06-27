---
type: {{ type }}
priority: {{ priority }}
source: {{ source }}
related_modules: {{ related_modules }}
created: {{ created_date }}
---
# {{ title }}

## 背景
{{ background }}

## 描述
{{ description }}

## 验收标准
{{#each acceptance_criteria}}
- [ ] {{ this }}
{{/each}}

## 附加信息
{{ additional_info }}

---
<!-- Variables:
  type        — feature | bugfix | enhancement | refactor | chore
  priority    — high | medium | low
  source      — cli | api | ide | webhook | manual
  related_modules — array of module names
  title       — requirement title
  background  — why this requirement is needed
  description — detailed description
  acceptance_criteria — list of verifiable criteria
  additional_info — links, screenshots, references
  created_date — ISO 8601 timestamp
-->
