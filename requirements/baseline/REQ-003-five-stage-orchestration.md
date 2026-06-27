---
id: REQ-003
title: 五阶段闭环编排
module: devloop-skills
status: baseline
inferred_from:
  - .claude/skills/devloop-reverse/SKILL.md
  - .claude/skills/devloop-resume/SKILL.md
  - design/02-detailed-design.md
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

# REQ-003: 五阶段闭环编排

## 推断来源

从以下代码反向推断：
- `.claude/skills/devloop-reverse/SKILL.md` (286 行)
- `.claude/skills/devloop-resume/SKILL.md` (128 行)
- `design/02-detailed-design.md` (完整五阶段设计)
- 23 个已有 Skill 定义（`.claude/skills/*/SKILL.md`）

## 功能描述

系统应提供从代码逆向分析到功能交付的完整五阶段闭环编排能力：

1. **阶段一 — 逆向分析 (reverse)**: 代码扫描 → 并行模块分析 → KB 汇总 → 基线需求反推 → 门禁校验 → 提交。复用 codegraph MCP + dispatching-parallel-agents + brainstorming
2. **阶段二 — 需求摄入 (intake)**: 输入解析 → KB 上下文加载 → 影响分析 → NFR 强制评估 → 需求文档生成 → 门禁校验。复用 codegraph MCP + brainstorming + openspec-propose
3. **阶段三 — 代码生成 (build)**: 需求拉取 → 方案设计 (brainstorming) → 实现计划 (writing-plans) → 执行实现 (sdd 或 executing-plans) → 门禁校验
4. **阶段四 — 验证与交付 (verify)**: 自动化验证 → 代码审查 → 修复循环（≤3次）→ 收尾合并 → 需求状态更新
5. **阶段五 — 闭环回写 (loop)**: 扫描待处理队列 → 优先级排序 → 路由到阶段二/三 → 循环直到队列空

## 当前实现

- 编排 Skill（已实现）:
  - devloop-reverse (286 行) — 阶段一
  - devloop-resume (128 行) — 中断恢复
- 待实现: devloop-intake, devloop-build, devloop-verify, devloop-loop
- 编排模式: 薄编排层，不重复实现已有 Skill 功能
- 确认级别: Level 1（快速通道）、Level 2（标准）、Level 3（高保障）
- 优先级算法: priority_score（high=100/medium=50/low=10）+ dependency_score
