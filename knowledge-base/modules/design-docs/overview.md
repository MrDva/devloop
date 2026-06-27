---
module: design-docs
path: design/
type: config
source: auto-generated
confidence: 0.80
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - design/00-README.md
  - design/01-overview.md
  - design/02-detailed-design.md
  - design/03-flowcharts.md
  - design/05-implementation-tasks.md
---

# design-docs — 模块概述

## 职责

记录 DevLoop (Closed-Loop Development System) 系统的完整设计——从概念概述到详细设计、流程图、实现任务清单。设计文档是整个 DevLoop 系统的**配置层和规范源头**，所有实现阶段的编排 Skill、脚本、模板均以此设计为蓝本。

## 模块类型

config

## 关键指标

- 文件数: 6 (00-README.md, 01-overview.md, 02-detailed-design.md, 03-flowcharts.md, 04-interactive-ui.html, 05-implementation-tasks.md)
- 主要语言: Markdown + YAML frontmatter + Mermaid
- 估计 Token 数: 35000

## 核心功能

- **架构定义**: 定义 DevLoop 的五阶段闭环架构（逆向分析 → 需求摄入 → 代码生成 → 验证交付 → 闭环回写）
- **门禁体系**: 设计 18 个质量门禁（含硬/软分离策略）的规则和判定方式
- **Skill 编排**: 定义 6 个新建编排 Skill 委托 14 个已有 Skill 的调用关系
- **数据模型**: 定义 loop-state.yaml、guard-config.yaml、KB manifest 等核心数据结构
- **KB 防御体系**: 设计知识库的三级信任体系、漂移检测、保鲜触发机制
- **中断恢复**: 设计会话中断/上下文压缩恢复的完整流程
- **确认策略**: 设计三级确认体系（Level 1-3），默认高保障

## 对外接口（实现状态标注: ✅=已实现, 📋=设计中, ⏳=待实现）

- ✅ `devloop-reverse` — 阶段一总编排 Skill（Phase 1 已交付）
- ✅ `devloop-resume` — 中断恢复 Skill（Phase 0 已交付）
- 📋 `devloop-intake` — 阶段二总编排 Skill（设计完成，待 Phase 2 实现）
- 📋 `devloop-build` — 阶段三总编排 Skill（设计完成，待 Phase 3 实现）
- 📋 `devloop-verify` — 阶段四总编排 Skill（设计完成，待 Phase 4 实现）
- 📋 `devloop-loop` — 阶段五总编排 Skill（设计完成，待 Phase 4 实现）
- ✅ `comet-state` — 状态管理脚本（Phase 0 已交付）
- ✅ `devloop-guard` — 门禁执行引擎脚本（Phase 0 已交付）
- ⏳ `kb-drift-check` — KB 漂移检测脚本（待 Phase 5 实现）
- ⏳ `kb-freshness-trigger` — KB 保鲜触发脚本（待 Phase 5 实现）
- ⏳ `kb-trust-update` — KB 信任级别评估脚本（待 Phase 5 实现）
- ⏳ `confirmation-queue` — 确认队列管理脚本（待 Phase 5 实现）

## 依赖关系

- **01-overview.md**: 提供给所有人阅读的总体设计，包含系统愿景、架构概览、ADR
- **02-detailed-design.md**: 提供给实现者的详细设计，包含每阶段 Skill 链、门禁规则、模板、中断恢复、KB 防御等 18 个章节
- **03-flowcharts.md**: 提供给所有人的 Mermaid 流程图，包含系统全景图、各阶段流程图、状态机、序列图
- **04-interactive-ui.html**: 交互式可视化 HTML，用于演示和评审
- **05-implementation-tasks.md**: 提供给实现者的任务清单，36 个任务分 6 个 Phase

## 被依赖关系

- **devloop-reverse**: 参考 02 第 1 节和 05 Phase 1 实现
- **devloop-intake**: 参考 02 第 2 节和 05 Phase 2 实现
- **devloop-build**: 参考 02 第 3 节和 05 Phase 3 实现
- **devloop-verify**: 参考 02 第 4 节和 05 Phase 4 实现
- **devloop-loop**: 参考 02 第 5 节和 05 Phase 4 实现
- **devloop-resume**: 参考 02 第 8 节和 05 Phase 0 实现
- **comet-state**: 参考 02 第 8.2 节实现
- **devloop-guard**: 参考 02 第 7 节和 05 Phase 0 实现
- **所有新建脚本**: 均以 02 对应章节和 05 对应任务为蓝本

## 备注

设计文档 v3 的核心改进：门禁硬/软分离（防止空洞阻断）、基线需求用途限制（reference_only，防止低质逆向污染下游）、阶段二插入 NFR 强制评估步骤（防止非功能需求断层）。设计预留了并发扩展点（loop-lock.yaml），但 Phase 1 仅支持单开发者串行执行。

---
<!-- Variables:
  module_name — design-docs
  module_path — design/
  module_type — config
  confidence — 0.95
  drift_score — 0.0
  last_updated — 2026-06-27T14:00:00Z
  generated_from — [design/00-README.md, design/01-overview.md, design/02-detailed-design.md, design/03-flowcharts.md, design/05-implementation-tasks.md]
  responsibility — 记录 DevLoop 系统的完整设计——从概念概述到详细设计、流程图、实现任务清单
  file_count — 6
  language — Markdown + YAML frontmatter + Mermaid
  token_estimate — 35000
  core_functions — [架构定义, 门禁体系, Skill编排, 数据模型, KB防御体系, 中断恢复, 确认策略]
  public_apis — [devloop-reverse, devloop-intake, devloop-build, devloop-verify, devloop-loop, devloop-resume, comet-state, devloop-guard, kb-drift-check, kb-freshness-trigger, kb-trust-update, confirmation-queue]
  dependencies — [01-overview.md, 02-detailed-design.md, 03-flowcharts.md, 04-interactive-ui.html, 05-implementation-tasks.md]
  dependents — [devloop-reverse, devloop-intake, devloop-build, devloop-verify, devloop-loop, devloop-resume, comet-state, devloop-guard, 所有新建脚本]
  notes — 设计文档v3核心改进：门禁硬/软分离、基线需求用途限制(reference_only)、NFR强制评估。预留并发扩展点，Phase 1仅单开发者串行。
-->
