---
analysis_timestamp: {{ analysis_timestamp }}
schema_version: "2.0"
method: multi-signal-clustering-v2
project_type: {{ project_type }}
seed_strategy: {{ seed_strategy }}
classification_system: {{ classification }}
phases:
  - api-seed
  - entity-anchor
  - call-graph-expansion
  - import-proximity
  - naming-patterns
  - merge-and-dedup
  - classify
total_business_functions: {{ total_bfs }}
total_files_analyzed: {{ total_files }}
orphan_files: {{ orphan_files }}
clustering_quality_score: {{ quality_score }}
confidence_distribution:
  high: {{ confidence.high }}
  medium: {{ confidence.medium }}
  low: {{ confidence.low }}
---

# 业务功能聚类报告

## 项目概述
- **项目类型**: {{ project_type_label }}
- **语言**: {{ language }}
- **框架**: {{ framework }}
- **种子策略**: {{ seed_strategy_label }}
- **分类体系**: {{ classification_label }}

## 项目检测依据
| 维度 | 检测信号 | 匹配 |
|------|---------|------|
{{#each detection_signals}}
| {{ this.dimension }} | {{ this.signal }} | {{ this.match }} |
{{/each}}

**判定结果**: {{ project_type }}
{{#if project_type == "generic"}}
> ⚠️ **降级为 generic 模式** — 未匹配到已知项目类型。使用改进版目录扫描 + 调用图/命名信号子聚类。
{{/if}}

## 聚类过程

### Phase 1: 种子收集
- **策略**: {{ seed_strategy_label }}
- **提取种子数**: {{ phase1.seed_count }}
- **种子来源**:
{{#each phase1.seeds}}
  - `{{ this.source }}` → `{{ this.handler }}`
{{/each}}

### Phase 2: 实体锚点识别
- **识别实体数**: {{ phase2.entity_count }}
{{#each phase2.entities}}
  - `{{ this.name }}` ({{ this.kind }}) — 被 {{ this.file_count }} 个文件操作
{{/each}}

### Phase 3: 调用图扩展
- **扩展深度**: {{ phase3.max_depth }} 层
- **纳入符号数**: {{ phase3.symbol_count }}
- **纳入文件数**: {{ phase3.file_count }}

### Phase 4: 导入邻近度
- **相似度阈值**: {{ phase4.similarity_threshold }}
- **文件对分析数**: {{ phase4.pair_count }}
- **发现的凝聚组**: {{ phase4.group_count }}

### Phase 5: 命名约定
- **识别的命名组**: {{ phase5.prefix_count }}
{{#each phase5.prefixes}}
  - `{{ this.prefix }}` → {{ this.symbol_count }} 个符号 → 候选归属: {{ this.candidate_bf }}
{{/each}}

### Phase 6: 合并与去重
- **候选聚类数** (合并前): {{ phase6.candidate_count }}
- **最终业务功能数** (合并后): {{ phase6.final_count }}
- **加权阈值**: {{ phase6.threshold }}
- **共享文件数** (归属 >1 功能): {{ phase6.shared_file_count }}

### Phase 7: 分类
- **分类体系**: {{ classification_label }}
- **分布**:
{{#each phase7.distribution}}
  - {{ this.category }}: {{ this.count }}
{{/each}}

## 聚类结果总览

| 业务功能 | 类型 | 置信度 | 专属文件 | 共享文件 | 总文件 | API/入口 |
|---------|------|--------|---------|---------|--------|---------|
{{#each business_functions}}
| {{ this.name }} | {{ this.category }} | {{ this.confidence }} | {{ this.size.primary_files }} | {{ this.size.shared_files }} | {{ this.size.total_files }} | {{ this.entry_count }} |
{{/each}}

## 共享基础
{{#each shared_foundations}}
- **{{ this.name }}**: {{ this.description }} — 被 {{ this.used_by_count }} 个功能使用
{{/each}}

## 孤儿文件
{{#if orphan_files > 0}}
以下文件未被分配到任何业务功能（可能为配置、类型定义或新的独立功能）:
{{#each orphan_file_list}}
- `{{ this.path }}` — 原因: {{ this.reason }}
{{/each}}
{{else}}
✅ 所有源文件均已分配到至少一个业务功能。
{{/if}}

## 低置信度功能
{{#if low_confidence_count > 0}}
以下业务功能的聚类置信度 < 0.70，建议人工审核:
{{#each low_confidence_bfs}}
- **{{ this.name }}** ({{ this.confidence }}) — 问题: {{ this.issue }}
{{/each}}
{{else}}
✅ 所有业务功能的聚类置信度 >= 0.70。
{{/if}}

## 质量评估
- **聚类质量分数**: {{ quality_score }} / 1.0
- **文件覆盖率**: {{ file_coverage }}%
- **入口覆盖率**: {{ entry_coverage }}%
- **孤儿文件数**: {{ orphan_files }}

{{#if quality_score < 0.80}}
> ⚠️ 聚类质量低于阈值 0.80。建议:
> 1. 检查 Phase 1 种子收集是否完整
> 2. 尝试降低 Phase 6 加权阈值（当前 {{ phase6.threshold }}）
> 3. 手动指定额外的入口种子
{{/if}}

---
<!-- Variables:
  analysis_timestamp — ISO 8601
  project_type — web-api | cli | frontend | library | data-pipeline | generic
  seed_strategy — api-route | cli-command | page-route | public-export | data-flow | directory-fallback
  classification — 分类体系标识
  total_bfs, total_files, orphan_files, quality_score
  confidence — {high, medium, low} counts
  detection_signals — [{dimension, signal, match}]
  phase1 — {seed_count, seeds[]}
  phase2 — {entity_count, entities[]}
  phase3 — {max_depth, symbol_count, file_count}
  phase4 — {similarity_threshold, pair_count, group_count}
  phase5 — {prefix_count, prefixes[]}
  phase6 — {candidate_count, final_count, threshold, shared_file_count}
  phase7 — {distribution[]}
  business_functions — summary list
  shared_foundations — [{name, description, used_by_count}]
  orphan_file_list — [{path, reason}]
  low_confidence_bfs — [{name, confidence, issue}]
  file_coverage, entry_coverage
-->
