---
module: {{ module_name }}
path: {{ module_path }}
type: {{ module_type }}
source: auto-generated
confidence: {{ confidence }}
drift_score: {{ drift_score }}
last_updated: {{ last_updated }}
generated_from:
{{#each generated_from}}
  - {{ this }}
{{/each}}
---

# {{ module_name }} — 模块概述

## 职责
{{ responsibility }}

## 模块类型
{{ module_type }}

## 关键指标
- 文件数: {{ file_count }}
- 主要语言: {{ language }}
- 估计 Token 数: {{ token_estimate }}

## 核心功能
{{#each core_functions}}
- **{{ this.name }}**: {{ this.summary }}
{{/each}}

## 对外接口
{{#each public_apis}}
- `{{ this.signature }}` — {{ this.description }}
{{/each}}

## 依赖关系
{{#each dependencies}}
- **{{ this.module }}**: {{ this.relationship }}
{{/each}}

## 被依赖关系
{{#each dependents}}
- **{{ this.module }}**: {{ this.relationship }}
{{/each}}

## 备注
{{ notes }}

---
<!-- Variables:
  module_name — module name
  module_path — filesystem path to module
  module_type — service | gateway | library | utility | ui | data | config | other
  confidence — 0.0 - 1.0 (auto-generated confidence)
  drift_score — 0.0 - 1.0 (drift from current code)
  last_updated — ISO 8601 timestamp
  generated_from — list of source files
  responsibility — what this module is responsible for
  file_count — number of files in module
  language — primary programming language
  token_estimate — estimated token count for KB entry
  core_functions — list of {name, summary}
  public_apis — list of {signature, description}
  dependencies — list of {module, relationship}
  dependents — list of {module, relationship}
  notes — additional notes
-->
