---
id: REQ-004
title: KB 模板系统
module: kb-templates
status: baseline
inferred_from:
  - templates/kb-module-overview.md
  - templates/kb-module-api.md
  - templates/kb-module-data-model.md
  - templates/kb-module-business-logic.md
  - templates/kb-module-dependencies.md
  - templates/requirement-doc.md
  - templates/nfr-checklist.md
confidence: high
source: auto-inferred
usage_restriction: reference_only
created: "2026-06-27T14:00:00Z"
---

> ⚠️ **AUTO-INFERRED BASELINE** | confidence: high | source: code reverse-engineering
>
> This requirement was inferred from existing code implementation, NOT from original design documents or specifications. It may not reflect original design intent.
>
> **Usage restriction (`reference_only`)**: This baseline requirement serves as **documentation reference only**. It MUST NOT be used as a mandatory constraint when designing new features in Stage 3. If a new requirement conflicts with a baseline requirement, the baseline requirement gives way.

# REQ-004: KB 模板系统

## 推断来源

从以下代码反向推断：
- `templates/` 目录下 11 个模板文件
- 每个模板的 `<!-- Variables: -->` 变量清单

## 功能描述

系统应提供标准化的产出物模板系统，支持：

1. **KB 条目模板（5 个）**: overview(模块概述)、api(API 文档)、data-model(数据模型)、business-logic(业务逻辑)、dependencies(依赖关系)。每个模板定义具体的 frontmatter 字段和 Markdown 结构
2. **需求文档模板（2 个）**: requirement-input(简易需求输入，7 个字段)、requirement-doc(完整需求文档，20+ 个字段，含基线模式条件块)
3. **分析/验证模板（3 个）**: impact-analysis(影响分析)、nfr-checklist(非功能需求强制评估清单，7 个维度)、verification-checklist(验证检查清单)
4. **清单模板（1 个）**: kb-manifest.yaml(模块索引清单)
5. **变量占位符**: 使用 `{{ }}` Handlebars 风格语法，支持 `{{#each}}` 迭代和 `{{#if}}` 条件渲染
6. **基线需求条件块**: requirement-doc.md 包含 `status == baseline` 条件渲染，显示 ⚠️ AUTO-INFERRED BASELINE 警告和 `usage_restriction: reference_only` 声明
7. **NFR 评估框架**: nfr-checklist.md 定义 7 个维度（认证授权/输入校验/数据保护/速率限制/审计日志/性能/可访问性），每个维度有独立触发条件和评估流程

## 当前实现

- 11 个模板文件位于 `templates/`
- 模板消费: 由 LLM agent 读取模板 → 理解变量结构 → 从分析上下文填充 → 生成文档
- 每个模板底部 `<!-- Variables: -->` 声明所有变量名和用途
- 防御标记: KB 模板 frontmatter 含 `source: auto-generated`, `confidence`, `drift_score`
- 基线用途限制: requirement-doc.md 含 `usage_restriction: reference_only` 条件块
