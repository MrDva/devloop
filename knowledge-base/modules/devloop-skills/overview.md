---
module: devloop-skills
source: auto-generated
confidence: 0.85
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - .claude/skills/
---

# devloop-skills -- 模块概述

## 职责

DevLoop 五阶段编排体系（Reverse -> Intake -> Build -> Verify -> Loop）+ 通用软件工程能力的 Skill 集合。负责从代码逆向分析到功能开发、验证、归档的完整闭环流程编排。同时提供 brainstorming、TDD、code review、debugging 等工程支撑能力。

## 模块类型

service

## 关键指标

- 文件数: 64（27 个 Skill，每个含 SKILL.md + 辅助文件）
- 主要语言: Markdown（Skill 定义）
- 估计 Token 数: 80000

## 核心功能

- **DevLoop 编排层（6 个）**: devloop-reverse, devloop-intake, devloop-build, devloop-verify, devloop-loop, devloop-resume -- 五阶段闭环编排与中断恢复
- **Comet/OpenSpec 流程层（11 个）**: openspec-new-change, openspec-propose, openspec-explore, openspec-ff-change, openspec-continue-change, openspec-apply-change, openspec-verify-change, openspec-sync-specs, openspec-archive-change, openspec-bulk-archive-change, openspec-onboard -- 基于 OpenSpec CLI 的结构化变更管理（从创建到归档）
- **通用工程层（10 个）**: brainstorming, writing-plans, subagent-driven-development, executing-plans, test-driven-development, requesting-code-review, receiving-code-review, systematic-debugging, verification-before-completion, finishing-a-development-branch -- 需求澄清、计划制定、实现执行、代码审查、调试验证、分支收尾

## 对外接口

- `/devloop reverse` -- 触发阶段一逆向分析（全量或 `--scope <module>` 增量）
- `/devloop resume` -- 手动触发中断恢复
- `/devloop kb-fresh --mode fix` -- 手动触发 KB 保鲜
- `Skill` 工具加载各 Skill（如 `brainstorming`, `writing-plans`, `subagent-driven-development` 等）
- `/opsx:new`, `/opsx:propose`, `/opsx:apply`, `/opsx:verify`, `/opsx:archive` 等 -- OpenSpec 快捷命令
- `comet-state` / `devloop-guard` 脚本 -- 由编排 Skill 内部调用

## 依赖关系

- **comet-scripts 模块**: 提供 `comet-state` 状态管理脚本和 `devloop-guard` 门禁校验脚本
- **kb-templates 模块**: 提供 `templates/kb-module-*.md` 系列模板
- **codegraph 模块**: 提供 MCP 工具 `codegraph_explore`（全项目扫描）和 `codegraph_node`（单文件/符号读取）
- **OpenSpec CLI**: 提供 `openspec` 命令用于变更生命周期管理

## 被依赖关系

- 无（Skill 是顶层编排，不被其他模块调用）

## 备注

所有 27 个 Skill 定义位于 `.claude/skills/<name>/SKILL.md`。每个 Skill 定义文件包含 YAML frontmatter（name, description）和 Markdown 正文（触发条件、流程步骤、门禁规则）。编排类 Skill（如 devloop-reverse）内部通过 `dispatching-parallel-agents` 调用其他 Skill 实现并行处理。

---
<!-- Variables:
  module_name -- devloop-skills
  module_path -- .claude/skills/
  module_type -- service
  confidence -- 0.85 (核心文件已完整读取，其他采用采样)
  drift_score -- 0.0
  last_updated -- 2026-06-27T14:00:00Z
  generated_from -- .claude/skills/
  responsibility -- DevLoop 五阶段编排体系 + 通用软件工程能力
  file_count -- 64
  language -- Markdown
  token_estimate -- 80000
  core_functions -- 27 个 Skill 分三层
  public_apis -- 四种触发方式
  dependencies -- 三个上游模块 + OpenSpec CLI
  dependents -- 无下游依赖
  notes -- Skill 是顶层编排方
-->
