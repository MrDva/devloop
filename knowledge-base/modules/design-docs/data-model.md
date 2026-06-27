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

# design-docs — 数据模型

## 1. loop-state.yaml

**存储位置**: `.comet/loop-state.yaml`
**用途**: 持久化 DevLoop 流程状态，用于中断恢复

```yaml
devloop:
  version: "1.0"               # 状态文件版本
  updated: "2026-06-27T10:30:00Z"

  current_stage: <1-5>         # 当前阶段编号
  current_step: <string>       # 当前步骤标识，如 "3.4"

  active_requirement:           # 当前处理的需求
    id: REQ-010                # 需求编号
    title: "添加 OAuth2 第三方登录"  # 需求标题
    change_name: add-oauth2-support  # OpenSpec change name

  checkpoint:                  # 检查点（中断恢复点）
    stage: <1-5>
    step: <string>
    last_action: <string>      # 最后一个操作描述
    timestamp: <ISO 8601>
    git_commit: <sha>          # 最后一个 commit

  completed_steps:             # 已完成步骤映射
    stage-1:
      - step: 1.1
        status: completed
        timestamp: "2026-06-27T09:00:00Z"
      - step: 1.2
        status: completed
        timestamp: "2026-06-27T09:15:00Z"

  gate_results:                # 门禁结果缓存
    G1.1: { passed: true, warnings: ["module coverage: 85%"] }
    G1.2: { passed: true }

  pending_confirmations:       # 待处理确认队列
    - type: plan_ready
      req: REQ-010
      message: "Implementation plan is ready for review"
      timestamp: "2026-06-27T10:30:00Z"

  errors:                      # 错误记录
    - step: 3.3
      error: "Design guard failed: missing delta spec"
      resolved: true
      resolution: "Re-ran brainstorming with explicit spec delta"
```

### 字段定义

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| devloop.version | string | Y | 状态文件版本号 |
| devloop.updated | ISO 8601 | Y | 最后更新时间 |
| devloop.current_stage | int (1-5) | Y | 当前阶段编号 |
| devloop.current_step | string | Y | 当前步骤标识 |
| devloop.active_requirement.id | string | Y | 需求唯一 ID (REQ-NNN) |
| devloop.active_requirement.title | string | Y | 需求标题 |
| devloop.active_requirement.change_name | string | N | OpenSpec change name |
| devloop.checkpoint.stage | int | Y | 检查点阶段 |
| devloop.checkpoint.step | string | Y | 检查点步骤 |
| devloop.checkpoint.last_action | string | Y | 最后操作描述 |
| devloop.checkpoint.timestamp | ISO 8601 | Y | 检查点时间 |
| devloop.checkpoint.git_commit | string | N | 关联 git commit |
| devloop.completed_steps | map | N | 已完成步骤索引 |
| devloop.gate_results | map | N | 门禁结果缓存 |
| devloop.pending_confirmations | array | N | 待处理确认列表 |
| devloop.errors | array | N | 错误列表 |

---

## 2. guard-config.yaml

**存储位置**: `templates/guard-config.yaml`
**用途**: 定义 18 个门禁规则的配置

### 门禁类型枚举

| 类型 | 判定方式 | 阻断权限 |
|------|---------|---------|
| hard | 脚本自动判定（`test -f`、`grep -c`、退出码等） | 可 BLOCK |
| soft | LLM 评估语义质量 | 仅 WARN（永不阻断） |
| mixed | 包含 hard + soft 两层，分别应用各自规则 | hard 可 BLOCK, soft 仅 WARN |

### on_fail 动作枚举

| 动作 | 说明 |
|------|------|
| block_and_retry | 阻断流程，自动重试（最多 3 次） |
| warn_and_continue | 记录警告，继续流程 |
| reject | 直接拒绝，返回给调用方 |
| pause_for_human | 暂停等待人工决策 |

### 完整门禁配置结构

```yaml
gates:
  stage-1-reverse:
    - id: G1.1
      name: "模块覆盖率"
      type: hard               # 硬门禁：脚本计数，无歧义
      checks:
        - "modules_analyzed / total_modules >= 0.8"
      on_fail: warn_and_continue

    - id: G1.2
      name: "KB 条目完整性"
      type: hard
      checks:
        - "all(module.has('overview.md') for module in modules)"
        - "all(module.has('api.md') for module in modules)"
      on_fail: block_and_retry

    - id: G1.3
      name: "KB 内部一致性"
      type: mixed
      hard:
        - "all_wiki_link_targets_exist(knowledge-base/)"
      soft:
        - "交叉引用指向的模块描述与引用上下文一致"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G1.4
      name: "需求反向覆盖率"
      type: hard
      checks:
        - "modules_covered_by_req / total_modules >= 0.9"
      on_fail: warn_and_continue

    - id: G1.5
      name: "阶段一总门禁"
      type: hard
      checks:
        - "G1.2 = pass AND G1.3.hard = pass AND (G1.1.soft_pass OR G1.1 = pass)"

  stage-2-intake:
    - id: G2.1
      name: "输入有效性"
      type: hard
      checks:
        - "input.title 非空"
        - "input.description 非空"
        - "input.type in [feature, bugfix, enhancement, refactor, chore]"
      on_fail: reject

    - id: G2.2
      name: "KB 上下文相关性"
      type: soft
      checks:
        - "匹配到的 KB 模块至少 1 个"
        - "匹配模块的关键词与需求描述语义相关"
      on_fail: warn_and_continue

    - id: G2.3
      name: "影响分析完整性"
      type: mixed
      hard:
        - "影响分析文档存在且非空"
        - "包含 '涉及模块' 章节"
        - "包含 'API变更' 或明确声明无变更"
        - "包含 '数据模型变更' 或明确声明无变更"
        - "包含 '非功能需求评估' 章节"
        - "所有 NFR 触发维度在需求文档验收标准中有对应 checklist 项"
      soft:
        - "涉及模块的分析深度足够"
        - "API/数据模型变更描述具体到函数/字段级别"
        - "NFR 评估各维度的补充验收标准具体可测"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G2.4
      name: "需求文档结构"
      type: hard
      checks:
        - "frontmatter 包含 id, title, type, priority, status, module"
        - "kb_context 引用至少 2 个（或声明理由为 new_module）"
        - "需求文档内容与 openspec proposal.md 一致"
      on_fail: block_and_retry

    - id: G2.5
      name: "阶段二总门禁"
      type: hard
      checks:
        - "G2.1 = pass AND G2.3.hard = pass AND G2.3-NFR = pass AND G2.4 = pass"

  stage-3-build:
    - id: G3.1
      name: "需求可构建性"
      type: hard
      checks:
        - "需求 status in [proposed, designed, planned]"
        - "需求 kb_context 中引用的所有文件仍然存在"
      on_fail: block_and_retry

    - id: G3.2
      name: "Design 阶段质量"
      type: mixed
      hard:
        - "design.md 存在且非空"
        - "delta spec 文件存在"
        - "comet-guard 返回 ALL CHECKS PASSED"
        - "设计方案未将任何 baseline 需求（usage_restriction: reference_only）作为强制约束"
        - "如果设计方案与某个 baseline 需求冲突，已明确记录偏离理由"
      soft:
        - "设计方案不违反 architecture/overview.md 中的架构约束"
        - "delta spec 覆盖影响分析中的所有涉及模块"
        - "API 契约变更与 KB 中记录的现有契约兼容或明确标注 breaking change"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G3.3
      name: "Plan 阶段质量"
      type: mixed
      hard:
        - "所有 task 有唯一编号"
        - "每个 task 关联到具体的 spec delta"
        - "无循环依赖"
      soft:
        - "task 粒度合理（预估每个 task < 200 行变更）"
        - "task 顺序符合依赖逻辑"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G3.4
      name: "Build 阶段质量"
      type: hard
      checks:
        - "所有 task checkbox 已勾选"
        - "测试覆盖率不低于变更前"
        - "无 linter 错误"
        - "devloop-guard build ALL CHECKS PASSED"
      on_fail: block_and_retry

  stage-4-verify:
    - id: G4.1
      name: "自动化验证"
      type: hard
      checks:
        - "所有测试通过（退出码=0）"
        - "构建成功（退出码=0）"
        - "Lint / Type Check 零错误（退出码=0）"
      on_fail: block_and_retry

    - id: G4.2
      name: "代码审查"
      type: mixed
      hard:
        - "审查报告存在"
        - "无 Critical 级别问题"
      soft:
        - "审查覆盖正确性/安全性/性能/可维护性/复用性五个维度"
        - "审查意见有具体代码位置引用"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G4.3
      name: "修复循环限制"
      type: hard
      checks:
        - "修复循环次数 < 3"
        - "每次修复后不引入新的 Critical 问题"
      on_fail: pause_for_human

    - id: G4.4
      name: "阶段四总门禁"
      type: hard
      checks:
        - "G4.1 = pass AND G4.2.hard = pass AND G4.3 未超限"
        - "需求文档中的所有验收标准全部满足"
        - "KB 一致性：代码变更未破坏 KB 中的 API 契约"
        - "Git 状态干净（无未暂存变更）"

  stage-5-loop:
    - id: G5.1
      name: "闭环状态一致性"
      type: hard
      checks:
        - "需求文档 frontmatter.status 与所在目录一致"
        - "loop-state.yaml 中的 active_requirement 指向有效需求"
        - "无孤儿需求"
```

---

## 3. KB Manifest 结构

**存储位置**: `knowledge-base/.manifest.yaml`
**用途**: KB 索引清单，支持按需加载

```yaml
modules:
  - name: <string>                 # 模块名称
    path: <string>                 # 模块文件系统路径，如 src/<module>/
    type: <enum>                   # service | gateway | library | utility | ui | data | config
    files: <int>                   # 模块包含的文件数
    dependencies: <string[]>       # 依赖的模块列表
    keywords: <string[]>           # 关键词表，用于索引匹配
    entities: <string[]>           # 数据实体名列表
    apis: <string[]>               # API 接口名列表
    total_tokens_estimate: <int>   # KB 条目总 token 估算
    summary_tokens_estimate: <int> # overview.md 单独 token 估算
```

### KB 条目信任标记

```yaml
# 每个 KB 条目 frontmatter 的防御字段
source: auto-generated | human-reviewed | hybrid
confidence: <0.0-1.0>            # 可信度
last_reviewed: <ISO 8601>        # 最后审核日期
reviewed_by: null | <string>     # 审核人
drift_score: <0.0-1.0>          # 代码漂移度（<0.1=健康）
generated_from:                   # 来源追溯
  - src/<file>.ts
```

### 三级信任体系

| 级别 | 条件 | 门禁行为 | 更新策略 |
|------|------|---------|---------|
| Trusted | human-reviewed + confidence >= 0.9 + drift_score < 0.05 | 正常通过，强依赖 | 仅人工或增量+人工审核 |
| Tentative | auto-generated + confidence >= 0.7 | 正常通过 + warning | 自动更新，标记 auto-updated |
| Untrusted | confidence < 0.7 或 drift_score > 0.2 | 阻断，须人工审核 | 阻止使用，强制重建 |

### 漂移报告结构

```yaml
# knowledge-base/.drift-report.yaml
module: auth-service
drift_score: 0.12
changes:
  - type: api_signature_changed | new_entity | removed_function
    kb_entry: <string>           # KB 中原记录
    current: <string>            # 代码中当前状态
    auto_fixable: true | false
```

---

## 4. 需求状态机

```
状态流转（共 8 个状态）:

                    ┌─────────┐
                    │  passed  │ (外部输入)
                    └────┬────┘
                         ↓
               ┌─────────────────┐
               │     draft       │ ← 阶段二输入解析完成
               └────────┬────────┘
                    ┌───┴───┐
                    ↓       ↓
            ┌──────────┐  rejected  (G2.1 失败)
            │ proposed  │ ← 阶段二完成，G2.5 通过
            └─────┬─────┘
              ┌───┴───┐
              ↓       ↓
         designed   blocked (G3.1 失败)
         (G3.2 通过)
              ↓
           planned   ← 阶段三 Plan 完成，G3.3 通过
              ↓
          building  ← 阶段三 Build 开始
              ↓
            built   ← 所有 Task 完成，G3.4 通过
              ↓
         verifying  ← 阶段四开始
              ↓
        ┌───┴───┐
        ↓       ↓
   completed    fixing (G4.1/4.2 失败)
   (G4.4 通过)    ↓
          ┌───────┴───────┐
          ↓               ↓
      verifying         blocked (连续 3 次失败)

状态与目录映射:
  baseline/    = status: baseline
  in-progress/ = status: proposed ~ verifying
  completed/   = status: completed
```

---

## 5. 确认级别定义

| 级别 | 编码 | 默认 | 确认点 | 禁止降级条件 |
|------|------|------|--------|-------------|
| Level 3 | L3 | 是（全局默认） | 需求内容 + 设计方案 + 实现计划 + 验证失败 | — |
| Level 2 | L2 | 否 | 设计方案 + 验证失败 | — |
| Level 1 | L1 | 否 | 仅验证失败 | — |

自动降级条件（仅当不违反禁止降级规则时）:
- type == 'bugfix' AND priority == 'low' → Level 1
- type == 'chore' → Level 1
- priority == 'medium' AND affected_modules <= 3 → Level 2

禁止降级条件:
- type == 'security' （安全相关）
- affected_modules > 5
- data_model_changes includes 'migration'

---

## 6. 上下文预算分配模型

```yaml
# 以 200K token 窗口为例
context_budget:
  system_instructions: 20K      # 系统指令 + Skill 定义
  requirement_doc: 5K           # 当前需求文档
  kb_entries: 80K               # KB 相关条目（索引按需加载）
  source_code: 50K              # 相关源代码（codegraph）
  generated_artifacts: 30K      # 生成产物（design/plan/task）
  buffer: 15K                   # 缓冲
```

---

## 7. KB 保鲜触发配置

```yaml
freshness_triggers:
  - type: post_verify
    scope: affected_modules_only
  - type: cron
    schedule: "0 2 * * 0"         # 每周日凌晨 2:00
    scope: full_scan
    action: report_only
  - type: cron
    schedule: "0 3 1 * *"         # 每月 1 日凌晨 3:00
    scope: full_scan
    action: auto_fix
  - type: manual
    command: "/devloop kb-fresh --scope <module|all> --mode <report|fix>"
```
