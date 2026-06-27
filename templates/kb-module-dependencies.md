---
module: {{ module_name }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ module_name }} — 依赖关系

## 上游依赖（本模块依赖的）
{{#each upstream}}
### {{ this.module }}
- **依赖类型**: {{ this.dependency_type }}
- **依赖内容**: {{ this.what }}
- **耦合度**: {{ this.coupling }}
- **接口**: `{{ this.interface }}`
{{/each}}

## 下游依赖（依赖本模块的）
{{#each downstream}}
### {{ this.module }}
- **依赖类型**: {{ this.dependency_type }}
- **依赖内容**: {{ this.what }}
- **耦合度**: {{ this.coupling }}
{{/each}}

## 依赖图
```
{{ dependency_graph }}
```

## 循环依赖
{{#if has_circular}}
⚠️ **检测到循环依赖**:
{{#each circular_deps}}
- {{ this }}
{{/each}}
{{else}}
✅ 未检测到循环依赖
{{/if}}

---
<!-- Variables:
  module_name — module name
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  upstream — list of {module, dependency_type, what, coupling, interface}
  downstream — list of {module, dependency_type, what, coupling}
  dependency_graph — ASCII/text graph representation
  has_circular — boolean
  circular_deps — list of descriptions
-->
