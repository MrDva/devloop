---
module: devloop-skills
source: auto-generated
confidence: 0.85
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - .claude/skills/
---

# devloop-skills -- API 文档

## 触发方式总览

Skill 的触发方式有两种：
1. **CLI 命令**（`/devloop <action>`）-- 仅限 DevLoop 编排类 Skill
2. **Skill 工具加载** -- 所有通用 Skill，加载后按 SKILL.md 定义执行

## DevLoop 编排层 API

### devloop-reverse
- **触发**: `/devloop reverse` 或 `/devloop reverse --scope modules=<module1>,<module2>`
- **前置依赖**: codegraph（MCP）、dispatching-parallel-agents、brainstorming、comet-state、devloop-guard
- **输入**:
  - `--scope modules=<list>`（可选）: 增量模式，指定模块
  - `--mode fix`（可选）: 保鲜模式
- **产出**:
  - `knowledge-base/.manifest.yaml` -- 模块清单
  - `knowledge-base/modules/<name>/` -- 每模块 5 个 KB 文件
  - `knowledge-base/architecture/` -- 全局架构文档
  - `knowledge-base/apis/internal.md` -- 内部 API 总览
  - `knowledge-base/data-models/global.md` -- 全局数据模型
  - `requirements/baseline/REQ-<NNN>-<slug>.md` -- 基线需求文档
- **内部调用链**: Step1: codegraph_explore -> Step2: dispatching-parallel-agents -> Step3: LLM 整合 -> Step4: brainstorming -> Step5: devloop-guard -> Step6: git commit

### devloop-intake ⏳ (设计完成，SKILL.md 尚未创建)
- **触发**: DevLoop 阶段二入口（设计目标，尚未实现）
- **职责**: 需求确认与优先级排序。将 baseline requirements + 新需求统一评估，确定 Level 1/2/3 级别，计算 priority_score，制定执行计划
- **前置依赖**: devloop-reverse 产物（KB + 基线需求）
- **实现状态**: 待 Phase 2 实现，当前仅有设计文档

### devloop-build ⏳ (设计完成，SKILL.md 尚未创建)
- **触发**: DevLoop 阶段三入口（设计目标，尚未实现）
- **职责**: 从 plan 开始执行实现。自动选择 build mode（subagent-driven-development / executing-plans），调度实现 agent，管理修复循环
- **前置依赖**: devloop-intake 产物（plan.md, tasks.md）
- **实现状态**: 待 Phase 3 实现，当前仅有设计文档

### devloop-verify ⏳ (设计完成，SKILL.md 尚未创建)
- **触发**: DevLoop 阶段四入口（设计目标，尚未实现）
- **职责**: 验证实现是否满足 spec，运行门禁，确认完备性
- **前置依赖**: devloop-build 产物（实现的代码）
- **实现状态**: 待 Phase 4 实现，当前仅有设计文档

### devloop-loop ⏳ (设计完成，SKILL.md 尚未创建)
- **触发**: DevLoop 阶段五入口（设计目标，尚未实现）
- **职责**: 决策是否进入下一轮 DevLoop 循环。如果有未覆盖需求或新的变更需求，回到 reverse 或 intake 阶段
- **前置依赖**: devloop-verify 结果
- **实现状态**: 待 Phase 4 实现，当前仅有设计文档

### devloop-resume
- **触发**: 自动（会话重启检测到 loop-state.yaml）/ `/devloop resume`
- **前置依赖**: comet-state 脚本、loop-state.yaml
- **输入**: 无（自动读取 .comet/loop-state.yaml）
- **恢复流程**:
  1. `comet-state check <name> <phase> --recover` 读取状态
  2. 快速上下文重建（读 proposal.md, loop-state.yaml, KB overview.md, memory 文件）
  3. 处理待确认项
  4. 按恢复动作继续（resume_from_step / fresh_start / resume_from_pending / gate_failures_present）
  5. 调用对应阶段 Skill

## Comet/OpenSpec 流程层 API

所有 OpenSpec Skill 依赖 `openspec CLI`，通过 `Skill` 工具或 `/opsx:<action>` 快捷命令触发。

| Skill | /opsx 命令 | 职责 | 产出 |
|-------|-----------|------|------|
| **openspec-new-change** | `/opsx:new` | 创建新的变更，初始化 chang 目录 | `openspec/changes/<name>/` + `.openspec.yaml` |
| **openspec-propose** | `/opsx:propose` | 一步生成 proposal + design + tasks | proposal.md, design.md, tasks.md |
| **openspec-explore** | `/opsx:explore` | 探索模式（思路梳理，不写代码） | 无固定产出（对话探索） |
| **openspec-ff-change** | `/opsx:ff` | 快速生成所有 artifact | 完整 artifact 套件 |
| **openspec-continue-change** | `/opsx:continue` | 创建下一个 artifact | 按流程推进 |
| **openspec-apply-change** | `/opsx:apply` | 实现 tasks 中的任务 | 代码实现 |
| **openspec-verify-change** | `/opsx:verify` | 验证实现匹配 artifact | 验证报告 |
| **openspec-sync-specs** | `/opsx:sync` | 将 delta spec 同步到主 spec | 更新主 spec 文件 |
| **openspec-archive-change** | `/opsx:archive` | 归档已完成变更 | 变更归档 |
| **openspec-bulk-archive-change** | `/opsx:bulk-archive` | 批量归档多个变更 | 多个变更归档 |
| **openspec-onboard** | `/opsx:onboard` | 引导式上手流程 | 教育体验 |

## 通用工程层 API

| Skill | 触发方式 | 职责 | 调用者 |
|-------|---------|------|--------|
| **brainstorming** | `Skill` 工具 | 探索用户意图、澄清需求、设计方案 | devloop-reverse(Step4), 用户直接 |
| **writing-plans** | `Skill` 工具 | 从 spec 编写实现计划 | 用户直接，devloop-loop |
| **subagent-driven-development** | `Skill` 工具 | 通过子 agent 并行执行 plan tasks | devloop-build（推荐模式） |
| **executing-plans** | `Skill` 工具 | 单会话执行 plan（无子 agent） | devloop-build（降级模式） |
| **test-driven-development** | `Skill` 工具 | 红-绿-重构 TDD 循环 | 通用工程 Skill(内嵌) |
| **requesting-code-review** | `Skill` 工具 | 派发审查子 agent 验证代码质量 | subagent-driven-development |
| **receiving-code-review** | `Skill` 工具 | 技术严谨地处理审查反馈 | 审查接收方 |
| **systematic-debugging** | `Skill` 工具 | 失败时执行根因分析 | devloop-build(修复循环) |
| **verification-before-completion** | `Skill` 工具 | 声明完成前执行验证门禁 | finishing-a-development-branch |
| **finishing-a-development-branch** | `Skill` 工具 | 提交/PR 选项引导 | executing-plans 等 |
| **using-git-worktrees** | `Skill` 工具 | 设置隔离工作空间 | build 阶段入口 |
| **using-superpowers** | `Skill` 工具 | 会话最优先加载的技能发现入口 | 所有会话 |
| **writing-skills** | `Skill` 工具 | 创建/编辑/验证 Skill 定义 | 元技能 |

## 内部 API（Skill 间调用关系）

### Skill -> Skill 调用链

```
devloop-reverse
  ├── Step1 -> codegraph_explore (MCP)
  ├── Step2 -> dispatching-parallel-agents (each agent uses codegraph_node via MCP)
  └── Step4 -> brainstorming

devloop-build
  ├── [SDD mode] -> subagent-driven-development
  │     ├── per-task -> requesting-code-review (spec compliance + code quality)
  │     └── on failure -> systematic-debugging
  ├── [EP mode] -> executing-plans
  │     └── after all tasks -> finishing-a-development-branch
  │           └── before claim -> verification-before-completion
  └── on fix cycle -> systematic-debugging

devloop-verify -> verification-before-completion

devloop-resume -> 根据 stage 调用 devloop-reverse/intake/build/verify/loop

writing-skills -> test-driven-development（必须前置理解）
```

---
