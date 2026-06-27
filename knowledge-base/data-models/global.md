---
source: auto-generated
confidence: 0.90
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - knowledge-base/modules/comet-scripts/data-model.md
  - knowledge-base/modules/devloop-skills/data-model.md
  - knowledge-base/modules/kb-templates/data-model.md
  - knowledge-base/modules/design-docs/data-model.md
---

# DevLoop 全局数据模型

## 1. 运行时状态: loop-state.yaml

```yaml
devloop:
  version: "1.0"
  change_name: "<change-name>"
  current_stage: <1-5>
  current_step: "<stage.step>"
  build_mode: "executing-plans | subagent-driven-development"
  tdd_mode: "on | off"
  isolation: "none | worktree"
  active_requirement:
    id: REQ-XXX
    title: "<title>"
    status: "<proposed|designed|planned|built|verified|completed>"
  checkpoint:
    stage: <1-5>
    step: "<stage.step>"
    last_action: "<description>"
    timestamp: "<ISO-8601>"
    git_commit: "<hash>"
  completed_steps:
    - stage: <1-5>
      step: "<stage.step>"
      timestamp: "<ISO-8601>"
  gate_results:
    G1.1: {result: pass, msg: "..."}
    G1.5: {result: fail, msg: "..."}
  pending_confirmations:
    - type: "<plan_ready|design_ready|...>"
      req: REQ-XXX
      message: "<description>"
      created: "<ISO-8601>"
  errors: []
```

## 2. 门禁配置: guard-config.yaml

```yaml
gates:
  stage-N:
    - id: G<stage>.<num>
      name: "<name>"
      type: "hard | soft | mixed"
      stage: <1-5>
      description: "<description>"
      checks:
        - type: "<file_count|file_exists|count|wiki_links_valid|gate_aggregate>"
          rule: "<rule expression>"
          detail: "<human-readable detail>"
      on_fail: "block_and_retry | warn_and_continue | reject | pause_for_human"
      # mixed 类型额外有:
      hard: [<hard checks>]
      soft: [<soft rules for LLM evaluation>]
```

**支持的 check type**:
- `file_count` — 文件/目录数比较
- `file_exists_per_module` — 每个模块目录下指定文件存在性
- `count` — 计数比较
- `wiki_links_valid` — `[target]` 引用目标存在性
- `gate_aggregate` — 前序门禁结果汇总

**硬/软门禁数量**: hard: 12+, soft/mixed: 6 | 总计: 18

## 3. KB 条目 Frontmatter

```yaml
---
module: "<module-name>"
path: "<source-path>"
type: "service | gateway | library | utility | ui | data | config | other"
source: "auto-generated | human-reviewed | hybrid"
confidence: <0.0-1.0>
drift_score: <0.0-1.0>
last_updated: "<ISO-8601>"
last_reviewed: "<ISO-8601 | null>"
reviewed_by: "<identifier | null>"
generated_from:
  - "<source-file-path>"
---
```

**三级信任体系**:

| 级别 | 条件 | 门禁行为 |
|------|------|---------|
| Trusted | `human-reviewed` + `>=0.9` + `<0.05` | 正常通过 |
| Tentative | `auto-generated` + `>=0.7` | 通过 + warning |
| Untrusted | `<0.7` 或 `>0.2` | 阻断，强制重建 |

## 4. 基线需求 Frontmatter

```yaml
---
id: REQ-<NNN>
title: "<title>"
module: "<module-name>"
status: baseline
inferred_from:
  - "<source-file-path>"
confidence: "high | medium | low"
source: auto-inferred
usage_restriction: reference_only
created: "<ISO-8601>"
---
```

## 5. 需求文档状态机

```
                    ┌──────────┐
                    │ proposed │ ← 阶段二产出
                    └────┬─────┘
                         │ 阶段三 brainstorm
                    ┌────▼─────┐
                    │ designed │
                    └────┬─────┘
                         │ 阶段三 writing-plans
                    ┌────▼─────┐
                    │ planned  │
                    └────┬─────┘
                         │ 阶段三 sdd/executing-plans
                    ┌────▼─────┐
                    │  built   │
                    └────┬─────┘
                         │ 阶段四 verify
                    ┌────▼─────┐
                    │ verified │
                    └────┬─────┘
                         │ 阶段四 archive
                    ┌────▼─────┐
                    │completed │
                    └──────────┘

  异常状态: blocked | rejected | deferred
```

## 6. 确认级别定义

| 级别 | 触发条件 | 确认点 | 适用场景 |
|------|---------|--------|---------|
| Level 1 | bugfix + low priority | 零确认直到阶段四 | 低风险修复 |
| Level 2 | enhancement + medium | Design 确认，Plan 自动 | 常规增强 |
| Level 3 | feature + high | Design + Plan 暂停确认 | 高保障需求 |

## 7. KB Manifest (knowledge-base/.manifest.yaml)

```yaml
generated_at: "<ISO-8601>"
generated_by: "devloop-reverse"
total_modules: <N>
modules:
  - name: "<module-name>"
    path: "<source-directory>"
    type: "<module-type>"
    files: <count>
    dependencies: [<dep-modules>]
    keywords: [<extracted>]
    entities: [<core-entities>]
    apis: [<public-api-summary>]
    total_tokens_estimate: <N>
    summary_tokens_estimate: <N>
```
