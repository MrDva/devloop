---
business_function: {{ business_function_name }}
category: {{ category }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ business_function_name }} — 依赖关系

## 上游依赖（本功能依赖的）
{{#each upstream}}
### {{ this.business_function }}
- **依赖类型**: {{ this.dependency_type }}
- **依赖内容**: {{ this.what }}
- **耦合度**: {{ this.coupling }}
- **接口**: `{{ this.interface }}`
{{/each}}

## 下游依赖（依赖本功能的）
{{#each downstream}}
### {{ this.business_function }}
- **依赖类型**: {{ this.dependency_type }}
- **依赖内容**: {{ this.what }}
- **耦合度**: {{ this.coupling }}
{{/each}}

## 依赖类型说明
| 类型 | 含义 |
|------|------|
| `call` | 函数调用关系 |
| `import` | 模块/文件导入关系 |
| `entity_share` | 共享数据实体（同一实体被多方读写） |
| `middleware_chain` | 中间件链式组合（web-api 类型特有） |
| `route_mount` | 路由挂载关系（web-api 类型特有） |
| `command_hierarchy` | 命令层级关系（cli 类型特有） |
| `component_composition` | 组件组合关系（frontend 类型特有） |
| `data_flow` | 数据流转关系（data-pipeline 类型特有） |

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
  business_function_name — semantic business function name
  category — classification label per project type
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  upstream — list of {business_function, dependency_type, what, coupling, interface}
  downstream — list of {business_function, dependency_type, what, coupling}
  dependency_graph — ASCII/text graph representation
  has_circular — boolean
  circular_deps — list of descriptions
-->
