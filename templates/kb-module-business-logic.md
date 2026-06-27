---
module: {{ module_name }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ module_name }} — 业务逻辑

## 核心流程
{{#each core_flows}}
### {{ this.name }}
- **触发条件**: {{ this.trigger }}
- **前置条件**: {{ this.preconditions }}
- **流程步骤**:
{{#each this.steps}}
  {{ this.index }}. {{ this.description }}
{{/each}}
- **后置条件**: {{ this.postconditions }}
- **异常处理**:
{{#each this.exceptions}}
  - {{ this.condition }} → {{ this.handling }}
{{/each}}
{{/each}}

## 业务规则
{{#each business_rules}}
- **{{ this.name }}**: {{ this.rule }}
{{/each}}

## 副作用
{{#each side_effects}}
- {{ this }}
{{/each}}

---
<!-- Variables:
  module_name — module name
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  core_flows — list of {name, trigger, preconditions, steps[], postconditions, exceptions[]}
  business_rules — list of {name, rule}
  side_effects — list of strings
-->
