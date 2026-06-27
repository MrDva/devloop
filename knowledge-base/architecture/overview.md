---
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - knowledge-base/modules/comet-scripts/
  - knowledge-base/modules/devloop-skills/
  - knowledge-base/modules/kb-templates/
  - knowledge-base/modules/design-docs/
---

# DevLoop 系统架构概述

## 架构风格

**编排式分层架构** — DevLoop 是一个基于 Claude Code Skill 系统的软件开发闭环工具。上层是 6 个编排 Skill，委托下层已有 Skill 和脚本执行具体工作。

## 层次结构

```
┌──────────────────────────────────────────┐
│          DevLoop 编排层 (6 Skills)        │  ← 用户交互面
│  reverse / intake / build / verify /     │
│  loop / resume                           │
├──────────────────────────────────────────┤
│         通用工程 Skill 层 (11 Skills)      │  ← 委托执行
│  brainstorming / writing-plans /         │
│  sdd / tdd / code-review / debugging ... │
├──────────────────────────────────────────┤
│         基础设施层 (2 Scripts)             │  ← 状态/门禁
│  comet-state / devloop-guard             │
├──────────────────────────────────────────┤
│         数据层                             │
│  templates (11) / guard-config.yaml /    │
│  loop-state.yaml / KB modules            │
└──────────────────────────────────────────┘
```

## 核心模块

| 模块 | 类型 | 职责 | 文件数 |
|------|------|------|--------|
| [[comet-scripts]] | utility | 状态管理 + 门禁引擎 | 3 |
| [[devloop-skills]] | service | 五阶段编排 + 工程能力 | 64 |
| [[kb-templates]] | config | 产出物格式定义 | 11 |
| [[design-docs]] | config | 系统设计规范 | 6 |

## 数据流

1. **阶段一 (reverse)**: 代码库 → codegraph 扫描 → 并行模块分析 → KB 条目 + 基线需求
2. **阶段二 (intake)**: 用户输入 → KB 上下文加载 → 影响分析 → NFR 评估 → 需求文档 + OpenSpec Change
3. **阶段三 (build)**: 需求文档 → brainstorming 设计 → writing-plans 计划 → sdd/executing-plans 执行 → 代码变更
4. **阶段四 (verify)**: 代码变更 → 自动化验证 → code review → debugging(如需) → 合并
5. **阶段五 (loop)**: 已完成需求归档 → 扫描队列 → 路由到阶段二/三处理下一需求

## 关键设计决策

- **薄编排层**: 6 个 DevLoop Skill 只做编排，不重复实现已有 Skill 的功能
- **硬/软门禁分离**: 硬门禁脚本自动判定可 BLOCK，软门禁 LLM 评估仅 WARN
- **基线需求 reference_only**: 从代码反推的需求仅供背景参考，不约束新设计
- **单模块 < 50K tokens**: KB 条目按模块拆分，确保上下文窗口够用
- **中断恢复**: 每步保存 checkpoint 到 loop-state.yaml，支持会话中断后恢复
