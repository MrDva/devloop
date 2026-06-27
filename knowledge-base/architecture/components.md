---
source: auto-generated
confidence: 0.90
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - knowledge-base/modules/comet-scripts/
  - knowledge-base/modules/devloop-skills/
  - knowledge-base/modules/kb-templates/
  - knowledge-base/modules/design-docs/
---

# DevLoop 组件拓扑

## 组件总览

```
                    ┌──────────────┐
                    │   用户/IDE    │
                    └──────┬───────┘
                           │ /devloop 命令 / Skill 工具
                           ▼
    ┌──────────────────────────────────────────────┐
    │              devloop-skills                   │
    │                                              │
    │  ┌─────────┐ ┌─────────┐ ┌─────────┐        │
    │  │ reverse │ │ intake  │ │ build   │        │
    │  └────┬────┘ └────┬────┘ └────┬────┘        │
    │       │           │           │              │
    │  ┌────┴───────────┴───────────┴────┐        │
    │  │         verify → loop           │        │
    │  └─────────────────────────────────┘        │
    │       │           │           │              │
    │  ┌────┴────┐ ┌────┴────┐ ┌───┴──────┐      │
    │  │resume   │ │brain-   │ │writing-  │      │
    │  │         │ │storming │ │plans     │      │
    │  └─────────┘ └─────────┘ └──────────┘      │
    │  ┌─────────┐ ┌─────────┐ ┌──────────┐      │
    │  │  sdd    │ │  tdd    │ │dispatch  │      │
    │  └─────────┘ └─────────┘ └──────────┘      │
    └──────────────┬───────────────────────────────┘
                   │ 调用
    ┌──────────────┴───────────────────────────────┐
    │           comet-scripts                      │
    │  ┌─────────────┐  ┌─────────────────┐        │
    │  │ comet-state │  │ devloop-guard   │        │
    │  └──────┬──────┘  └────────┬────────┘        │
    └─────────┼──────────────────┼─────────────────┘
              │ 读写             │ 检查
    ┌─────────┴──────────────────┴─────────────────┐
    │              数据层                           │
    │  loop-state.yaml  guard-config.yaml          │
    │  knowledge-base/  requirements/               │
    └──────────────────────────────────────────────┘
```

## 组件通信

| 调用方 | 被调用方 | 方式 | 频率 |
|--------|---------|------|------|
| devloop-reverse | codegraph_explore (MCP) | MCP 工具调用 | 阶段一 Step 1 |
| devloop-reverse | dispatching-parallel-agents | Skill 工具 | 阶段一 Step 2 |
| devloop-reverse | brainstorming | Skill 工具 | 阶段一 Step 4 |
| devloop-reverse | comet-state | Bash 脚本 | 每 Step |
| devloop-reverse | devloop-guard | Bash 脚本 | 阶段一 Step 5 |
| devloop-intake | codegraph_explore (MCP) | MCP 工具调用 | 阶段二 Step 2 |
| devloop-intake | brainstorming | Skill 工具 | 阶段二 Step 3 |
| devloop-intake | openspec-propose | Skill 工具 | 阶段二 Step 5 |
| devloop-intake | comet-state / devloop-guard | Bash 脚本 | 每 Step |
| devloop-build | brainstorming | Skill 工具 | 阶段三 Step 3 |
| devloop-build | writing-plans | Skill 工具 | 阶段三 Step 4 |
| devloop-build | subagent-driven-development | Skill 工具 | 阶段三 Step 5 |
| devloop-build | executing-plans | Skill 工具 | 阶段三 Step 5 (小需求) |
| devloop-verify | verification-before-completion | Skill 工具 | 阶段四 Step 1 |
| devloop-verify | code-review | 内置命令 | 阶段四 Step 2 |
| devloop-verify | systematic-debugging | Skill 工具 | 阶段四 Step 3 (如需) |
| devloop-verify | finishing-a-development-branch | Skill 工具 | 阶段四 Step 4 |
| devloop-loop | devloop-intake / devloop-build | Skill 工具 | 阶段五 Step 3 |

## OpenSpec 组件

11 个 `openspec-*` Skill 构成独立的变更生命周期管理子系统：

```
openspec-new-change → openspec-propose → openspec-explore
                                          ↓
                                     openspec-ff-change
                                     openspec-continue-change
                                          ↓
                                     openspec-apply-change
                                          ↓
                                     openspec-verify-change
                                          ↓
                                     openspec-sync-specs
                                          ↓
                                     openspec-archive-change
                                     openspec-bulk-archive-change
```

通过 `openspec-onboard` 进行初次初始化。
