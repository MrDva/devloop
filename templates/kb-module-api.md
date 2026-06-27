---
module: {{ module_name }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ module_name }} — API 文档

## 导出符号
{{#each exports}}
### {{ this.name }}
- **类型**: {{ this.kind }}
- **签名**: `{{ this.signature }}`
- **描述**: {{ this.description }}
- **参数**:
{{#each this.parameters}}
  - `{{ this.name }}` ({{ this.type }}): {{ this.description }}
{{/each}}
- **返回值**: {{ this.return_type }} — {{ this.return_description }}
- **异常**: {{ this.throws }}
- **示例**:
```{{ language }}
{{ this.example }}
```
{{/each}}

## 内部 API
{{#each internal_apis}}
### {{ this.name }}
- **签名**: `{{ this.signature }}`
- **用途**: {{ this.purpose }}
{{/each}}

---
<!-- Variables:
  module_name — module name
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  exports — list of {name, kind, signature, description, parameters[], return_type, return_description, throws, example}
  internal_apis — list of {name, signature, purpose}
  language — primary language
-->
