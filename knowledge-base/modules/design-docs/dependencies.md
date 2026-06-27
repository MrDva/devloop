---
module: design-docs
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - design/00-README.md
  - design/01-overview.md
  - design/02-detailed-design.md
  - design/03-flowcharts.md
  - design/05-implementation-tasks.md
---

# design-docs — 依赖关系

## 上游依赖（本模块依赖的）

设计文档是纯文档，无代码级上游依赖。它仅引用源代码结构（通过描述系统架构和代码目录），但不依赖于运行时组件。

| 依赖项 | 依赖类型 | 依赖内容 | 耦合度 |
|--------|---------|---------|-------|
| 无 | — | 设计文档是独立纯文档配置，不依赖任何外部系统 | 无 |

---

## 下游依赖（依赖本模块的）

整个 DevLoop 系统的所有实现组件均以本设计文档为规范和依据。

### 编排 Skill（6 个新建）

| 模块 | 依赖类型 | 依赖内容 | 耦合度 |
|------|---------|---------|-------|
| devloop-reverse | 参考实现 | 阶段一流程、Skill 调用链、门禁规则、产物格式见 02 第 1 节 | strong |
| devloop-intake | 参考实现 | 阶段二流程、NFR 评估、输入解析、影响分析见 02 第 2 节 | strong |
| devloop-build | 参考实现 | 阶段三流程、Design/Plan/Build 编排、Build 模式选择见 02 第 3 节 | strong |
| devloop-verify | 参考实现 | 阶段四流程、验证策略、修复循环见 02 第 4 节 + 第 15 节 | strong |
| devloop-loop | 参考实现 | 阶段五流程、队列扫描、路由逻辑见 02 第 5 节 | strong |
| devloop-resume | 参考实现 | 中断恢复流程、状态读取、恢复策略见 02 第 8 节 | strong |

### 脚本（6 个新建）

| 模块 | 依赖类型 | 依赖内容 | 耦合度 |
|------|---------|---------|-------|
| comet-state | 参考实现 | 状态文件结构、字段定义、check/recover 逻辑见 02 第 8.2 节 | strong |
| devloop-guard | 参考实现 | 门禁配置结构、硬/软分离策略、on_fail 动作见 02 第 7 节 + guard-config.yaml 模板 | strong |
| kb-drift-check | 参考实现 | 漂移检测算法、漂移报告结构、阻断规则见 02 第 13 节 | strong |
| kb-freshness-trigger | 参考实现 | 保鲜触发配置、Git 变更检测、保鲜流程见 02 第 16 节 | strong |
| kb-trust-update | 参考实现 | 三级信任体系、信任级别判定条件见 02 第 13.2 节 | strong |
| confirmation-queue | 参考实现 | 确认队列管理、确认点行为见 02 第 12 节 | strong |

### 模板（10 个 .md 文件）

| 模板 | 设计依据 | 耦合度 |
|------|---------|-------|
| requirement-input.md | 02 第 2.1 节输入解析格式 | strong |
| requirement-doc.md | 02 第 6 节需求文档模板 | strong |
| kb-module-overview.md | 02 第 1.2.2 节模块 KB 结构 | strong |
| kb-module-api.md | 同上 | strong |
| kb-module-data-model.md | 同上 | strong |
| kb-module-business-logic.md | 同上 | strong |
| kb-module-dependencies.md | 同上 | strong |
| impact-analysis.md | 02 第 2.3 节影响分析报告格式 | strong |
| nfr-checklist.md | 02 第 2.2.4 节 NFR 强制评估 | strong |
| verification-checklist.md | 02 第 15 节验证策略 | strong |

### 测试和验证

| 模块 | 依赖类型 | 依赖内容 | 耦合度 |
|------|---------|---------|-------|
| Phase 0 任务 (T0.0-T0.4) | 任务依据 | 目录结构、脚本、模板创建见 05 实现任务 | strong |
| Phase 1 任务 (T1.0-T1.4) | 任务依据 | 阶段一实现任务见 05 实现任务 | strong |
| Phase 2 任务 (T2.0-T2.4) | 任务依据 | 阶段二实现任务见 05 实现任务 | strong |
| Phase 3 任务 (T3.0-T3.9) | 任务依据 | 阶段三实现任务见 05 实现任务 | strong |
| Phase 4 任务 (T4.0-T4.5) | 任务依据 | 阶段四 + 五实现任务见 05 实现任务 | strong |

---

## 依赖图

```
[design-docs] (纯文档，无上游依赖)
    │
    ├──→ [devloop-reverse]     ┐
    ├──→ [devloop-intake]      │
    ├──→ [devloop-build]       ├── 6 个编排 Skill (新建)
    ├──→ [devloop-verify]      │
    ├──→ [devloop-loop]        │
    ├──→ [devloop-resume]      ┘
    │
    ├──→ [comet-state]         ┐
    ├──→ [devloop-guard]       │
    ├──→ [kb-drift-check]      ├── 6 个脚本 (新建)
    ├──→ [kb-freshness-trigger]│
    ├──→ [kb-trust-update]     │
    ├──→ [confirmation-queue]  ┘
    │
    ├──→ [10 个模板文件]        ── 定义产物格式
    │
    └──→ [05 实现任务清单]      ── 36 个 Task 分 6 个 Phase
```

---

## 设计文档内部引用关系

### 文档间引用

| 源文档 | 引用目标 | 引用类型 | 用途 |
|--------|---------|---------|------|
| 00-README.md | 01-overview.md | 导航 | 总体设计入口 |
| 00-README.md | 02-detailed-design.md | 导航 | 详细设计入口 |
| 00-README.md | 03-flowcharts.md | 导航 | 流程图入口 |
| 00-README.md | 04-interactive-ui.html | 导航 | 交互式可视化入口 |
| 00-README.md | 05-implementation-tasks.md | 导航 | 实现任务入口 |
| 01-overview.md | 02-detailed-design.md 第 12 节 | 引用 | 确认策略详细设计 |
| 01-overview.md | 02-detailed-design.md 第 17 节 | 引用 | 并发预留设计 |
| 02-detailed-design.md | 03-flowcharts.md | 引用 | 流程图辅助理解 |
| 05-implementation-tasks.md | 01-overview.md | 任务依据 | 各阶段编排参考总体设计 |
| 05-implementation-tasks.md | 02-detailed-design.md | 任务依据 | 各任务具体实现参考详细设计 |
| 05-implementation-tasks.md | 03-flowcharts.md | 任务依据 | 流程图验证实现正确性 |

### 核心对应关系: 02 详细设计 → 05 实现任务

| 02 章节 | 05 对应 Phase | Task 数量 | 说明 |
|---------|---------------|-----------|------|
| 第 1 节: 阶段一 | Phase 1 (T1.0-T1.4) | 5 | 逆向分析实现 |
| 第 2 节: 阶段二 | Phase 2 (T2.0-T2.4) | 5 | 需求摄入实现 |
| 第 3 节: 阶段三 | Phase 3 (T3.0-T3.9) | 10 | 代码生成实现 |
| 第 4+5 节: 阶段四+五 | Phase 4 (T4.0-T4.5) | 6 | 验证交付+闭环 |
| 第 7 节: 门禁体系 | Phase 0 (T0.2) | 1 | devloop-guard |
| 第 8 节: 中断恢复 | Phase 0 (T0.1) | 1 | loop-state + comet-state |
| 第 13 节: KB 防御 | Phase 1 (T1.3-T1.4) | 2 | 漂移检测 |
| 第 16 节: KB 保鲜 | Phase 4 (T4.6) | 1 | 定时触发 |
| 第 6 节: 模板系统 | Phase 0 (T0.1) | 1 | 10 个模板 |

---

## 已有 Skill 复用依赖关系（14 个）

```
已有 Skill / MCP 工具       调用的 DevLoop Skill
─────────────────────────   ────────────────────────
dispatching-parallel-agents  devloop-reverse
brainstorming                devloop-reverse (需求推断)
                             devloop-intake (影响分析)
                             devloop-build (方案设计)
openspec-propose             devloop-intake
writing-plans                devloop-build
subagent-driven-development  devloop-build
executing-plans              devloop-build
test-driven-development      (被 subagent-driven-development 内部调用)
requesting-code-review       (被 subagent-driven-development 内部调用)
verification-before-completion  devloop-verify
code-review (内置)           devloop-verify
systematic-debugging         devloop-verify
finishing-a-development-branch  devloop-verify
using-git-worktrees          devloop-build
using-superpowers            元技能 (启动时自动加载)

codegraph_explore (MCP)      devloop-reverse (代码扫描)
                             devloop-intake (KB 索引查询)
codegraph_node (MCP)         devloop-reverse (模块符号读取)
```

---

## 循环依赖

未检测到循环依赖。设计文档是纯文档配置，所有依赖为实现方向的单向引用。

---

## 备注

设计文档是 DevLoop 系统的**唯一设计源头**。所有新建组件（6 编排 Skill + 6 脚本 + 10 模板 + 36 实现任务）均直接引用此设计文档作为实现依据。已有 Skill 不修改，仅通过编排层调用。设计文档本身无任何代码依赖，确保其作为长期参考稳定性不受代码变更影响。
