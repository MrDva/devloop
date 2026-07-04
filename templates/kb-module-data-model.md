---
business_function: {{ business_function_name }}
category: {{ category }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ business_function_name }} — 数据模型

## 主实体定义
{{#each entities}}
### {{ this.name }}
- **类型**: {{ this.entity_type }}
- **描述**: {{ this.description }}
- **所有权**: {{ this.ownership }}   # owned | shared — 是否被多个业务功能共享
- **共享者**: {{#each this.shared_by}}{{ this }}{{#unless @last}}, {{/unless}}{{/each}}
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

## 次要实体（读取但不拥有）
{{#each secondary_entities}}
### {{ this.name }}
- **来源业务功能**: {{ this.owned_by }}
- **本功能中的用途**: {{ this.usage }}
- **操作类型**: {{ this.operation }}   # read_only | write | read_write
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
  business_function_name — semantic business function name
  category — classification label per project type
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  entities — list of {name, entity_type, description, ownership, shared_by[], fields[], constraints[], relations[]}
  secondary_entities — list of {name, owned_by, usage, operation}
  state_machines — list of {entity, diagram}
-->
