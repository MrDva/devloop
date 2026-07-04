---
business_function: {{ business_function_name }}
category: {{ category }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ business_function_name }} — 业务逻辑

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

## 共享文件角色声明
{{#each shared_files}}
- **`{{ this.file }}`**: {{ this.role }}
  - 同时被以下功能使用: {{#each this.also_in}}`{{ this }}` {{/each}}
{{/each}}

---
<!-- Variables:
  business_function_name — semantic business function name
  category — classification label per project type
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  core_flows — list of {name, trigger, preconditions, steps[], postconditions, exceptions[]}
  business_rules — list of {name, rule}
  side_effects — list of strings
  shared_files — list of {file, role, also_in[]}
-->
