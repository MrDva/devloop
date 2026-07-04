---
business_function: {{ business_function_name }}
category: {{ category }}               # core | admin | infrastructure | foundation
source: auto-generated
confidence: {{ confidence }}
clustering_signals:
  api_seed: {{ signals.seed }}
  call_graph: {{ signals.call_graph }}
  import_proximity: {{ signals.import_proximity }}
  naming: {{ signals.naming }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
generated_from:
{{#each primary_files}}
  - {{ this }}
{{/each}}
shared_with:
{{#each shared_files}}
  - file: "{{ this.file }}"
    role: "{{ this.role }}"
    also_in: [{{#each this.also_in}}{{ this }}{{#unless @last}}, {{/unless}}{{/each}}]
{{/each}}
---

# {{ business_function_name }} — 业务功能概述

## 职责
{{ responsibility }}

## 功能类型
**{{ category_label }}** — {{ category_description }}

> 分类体系由阶段一 1.2.1 项目感知选择：`web-api` → core/admin/infrastructure/foundation

## 入口端点
{{#if api_endpoints}}
{{#each api_endpoints}}
- **{{ this.method }} {{ this.path }}**: {{ this.summary }}
{{/each}}
{{else}}
{{#if commands}}
{{#each commands}}
- **{{ this.name }}**: {{ this.summary }}
{{/each}}
{{else}}
{{#if page_routes}}
{{#each page_routes}}
- **{{ this.path }}** → `{{ this.component }}`
{{/each}}
{{else}}
_无公开入口端点（内部功能或被其他功能调用）_
{{/if}}
{{/if}}
{{/if}}

## 核心符号
### 函数
{{#each core_symbols.functions}}
- `{{ this }}`
{{/each}}
### 类型
{{#each core_symbols.types}}
- `{{ this }}`
{{/each}}

## 操作的数据实体
- **主实体**: {{#each data_entities}}{{ this }}{{#unless @last}}, {{/unless}}{{/each}}
- **次要实体**: {{#each secondary_entities}}{{ this }}{{#unless @last}}, {{/unless}}{{/each}}

## 文件归属
### 专属文件（{{ size.primary_files }} 个）
{{#each primary_files}}
- `{{ this }}`
{{/each}}
### 共享文件（{{ size.shared_files }} 个）
{{#each shared_files}}
- `{{ this.file }}` — {{ this.role }}
  - 同时归属于: {{#each this.also_in}}`{{ this }}` {{/each}}
{{/each}}

## 依赖关系
### 本功能依赖
{{#each depends_on}}
- **{{ this.business_function }}**: {{ this.reason }}
{{/each}}
### 被以下功能依赖
{{#each depended_by}}
- **{{ this.business_function }}**: {{ this.reason }}
{{/each}}

## 聚类证据
| 信号 | 命中 | 说明 |
|------|------|------|
| 入口种子 | {{ signals.seed }} | 是否有 API 路由/CLI 命令等入口点 |
| 调用图 | {{ signals.call_graph }} | 函数调用链是否形成紧密耦合 |
| 导入相似度 | {{ signals.import_proximity }} | 文件导入模式是否相似 |
| 命名约定 | {{ signals.naming }} | 符号命名是否有共享前缀/后缀 |

## 规模指标
- 专属文件: {{ size.primary_files }}
- 共享文件: {{ size.shared_files }}
- 总计(去重): {{ size.total_files }}
- KB 条目 Token 估算: {{ size.kb_tokens_estimate }}

## 备注
{{ notes }}

---
<!-- Variables:
  business_function_name — semantic name (kebab-case)
  category — classification label per project type classification system
  category_label — human-readable category label
  category_description — what this category means
  confidence — 0.0 - 1.0
  signals — {seed, call_graph, import_proximity, naming} each boolean
  drift_score — 0.0 - 1.0
  last_updated — ISO 8601 timestamp
  primary_files — list of files uniquely belonging to this bf
  shared_files — list of {file, role, also_in[]}
  responsibility — what this business function is responsible for
  api_endpoints — list of {method, path, summary} (web-api type)
  commands — list of {name, summary} (cli type)
  page_routes — list of {path, component} (frontend type)
  core_symbols — {functions[], types[]}
  data_entities — primary entity names
  secondary_entities — secondary entity names
  depends_on — list of {business_function, reason}
  depended_by — list of {business_function, reason}
  size — {primary_files, shared_files, total_files, kb_tokens_estimate}
  notes — additional notes
-->
