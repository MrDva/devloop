---
module: {{ module_name }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ module_name }} — 数据模型

## 实体定义
{{#each entities}}
### {{ this.name }}
- **类型**: {{ this.entity_type }}
- **描述**: {{ this.description }}
- **字段**:
{{#each this.fields}}
  - `{{ this.name }}` ({{ this.type }}){{#if this.required}} **必需**{{/if}}: {{ this.description }}
{{/each}}
- **约束**:
{{#each this.constraints}}
  - {{ this }}
{{/each}}
- **关联**:
{{#each this.relations}}
  - → {{ this.target }}: {{ this.type }} ({{ this.description }})
{{/each}}
{{/each}}

## 状态转换
{{#each state_machines}}
### {{ this.entity }}
```
{{ this.diagram }}
```
{{/each}}

---
<!-- Variables:
  module_name — module name
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  entities — list of {name, entity_type, description, fields[], constraints[], relations[]}
  state_machines — list of {entity, diagram}
-->
