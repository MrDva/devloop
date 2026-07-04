---
business_function: {{ business_function_name }}
category: {{ category }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
---

# {{ business_function_name }} — API 文档

## 公开入口
{{#if api_endpoints}}
{{#each api_endpoints}}
### {{ this.method }} {{ this.path }}
- **处理器**: `{{ this.handler }}`
- **认证要求**: {{ this.auth_required }}
- **描述**: {{ this.summary }}
- **请求参数**:
{{#each this.parameters}}
  - `{{ this.name }}` ({{ this.type }}, {{ this.in }}): {{ this.description }}{{#if this.required}} **必需**{{/if}}
{{/each}}
- **响应**: {{ this.response_type }}
- **错误码**:
{{#each this.error_codes}}
  - `{{ this.code }}`: {{ this.description }}
{{/each}}
{{/each}}
{{else}}
{{#if commands}}
{{#each commands}}
### {{ this.name }}
- **处理器**: `{{ this.handler }}`
- **描述**: {{ this.summary }}
- **参数/选项**:
{{#each this.options}}
  - `--{{ this.name }}` ({{ this.type }}): {{ this.description }}{{#if this.required}} **必需**{{/if}}
{{/each}}
{{/each}}
{{else}}
{{#if page_routes}}
{{#each page_routes}}
### {{ this.path }}
- **组件**: `{{ this.component }}`
- **描述**: {{ this.summary }}
- **Props**:
{{#each this.props}}
  - `{{ this.name }}` ({{ this.type }}): {{ this.description }}
{{/each}}
{{/each}}
{{else}}
_此业务功能无公开入口端点（内部使用或被其他功能调用）。_
{{/if}}
{{/if}}
{{/if}}

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
  business_function_name — semantic business function name
  category — classification label per project type
  confidence — 0.0 - 1.0
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  api_endpoints — list of {method, path, handler, auth_required, summary, parameters[], response_type, error_codes[]} (web-api)
  commands — list of {name, handler, summary, options[]} (cli)
  page_routes — list of {path, component, summary, props[]} (frontend)
  exports — list of {name, kind, signature, description, parameters[], return_type, return_description, throws, example}
  internal_apis — list of {name, signature, purpose}
  language — primary language
-->
