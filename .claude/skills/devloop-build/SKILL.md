---
name: devloop-build
description: "DevLoop 阶段三：代码生成 — 拉需求→委托设计→委托计划→委托执行→门禁→提交。"
---

# DevLoop Build — 阶段三：代码生成

根据阶段二产出的需求文档，走 Design → Plan → Build 完整流程，产出代码变更并提交。

## 触发条件

1. 用户在需求确认后触发：`/devloop build REQ-<NNN>`
2. `devloop-loop` 路由 `designed` 或 `planned` 状态的需求到此阶段
3. 用户手动指定跳过某些步骤：`/devloop build REQ-<NNN> --from plan`

## 流程总览

```
Step 1: 需求拉取        → (编排层 LLM)                              → 确定目标需求 + 确认级别
Step 2: 方案设计        → brainstorming                            → openspec/changes/<name>/design.md
Step 3: 实现计划        → writing-plans                             → openspec/changes/<name>/plan.md + tasks.md
Step 4: 代码实现        → subagent-driven-development (大需求)      → 代码变更 + 测试
                         或 executing-plans (小需求)
Step 5: 门禁校验         → devloop-guard check stage-3              → G3.1 - G3.4
Step 6: Git 提交         → git commit                                → [devloop] stage:3 action:build req:<id>
```

每个 Step 结束后调用 `comet-state checkpoint` 保存断点，支持中断恢复。

---

## Step 1: 需求拉取

**目标**: 确定本次构建的目标需求及其确认级别。

### 1.1 确定目标需求

若用户已指定 `REQ-<NNN>`，直接加载该需求文档。

否则扫描 `requirements/in-progress/` 目录，按优先级排序选取最高分需求：

```
priority_score = {
  high:   3
  medium: 2
  low:    1
}[priority]

dependency_score = |completed_dependencies| / |total_dependencies|  （无依赖时 = 1.0）

composite_score = priority_score * 0.7 + dependency_score * 0.3
```

选取 `composite_score` 最高且 `status != blocked` 的需求。

### 1.2 加载需求上下文

从需求文档 frontmatter 提取：
- `id`, `title`, `type`, `priority`, `status`
- `confirmation_level` (L1/L2/L3)
- `kb_context` (关联的 KB 条目路径列表)
- `affected_modules`, `api_changes`, `data_model_changes`
- `acceptance_criteria`（含 NFR 补充验收项）
- `impact_analysis` 引用

### 1.3 确认点预判

根据确认级别（记录在需求文档 frontmatter 中）确定暂停点：

| 级别 | Step 2 (Design) 后 | Step 3 (Plan) 后 |
|------|-------------------|-------------------|
| Level 1 | ❌ 不暂停 | ❌ 不暂停 |
| Level 2 | ✅ 暂停确认 | ❌ 自动继续 |
| Level 3 | ✅ 暂停确认 | ✅ 暂停确认 |

**门禁 G3.1** — 需求可构建性（hard）:
- 需求文档存在且 `status ∈ [proposed, designed, planned]`
- `acceptance_criteria` 非空（至少 1 条）
- `kb_context` 引用文件存在
- 需求文档 frontmatter 无 `blocked` 标记

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 3 3.1 "Requirement pulled: REQ-{NNN} (level={L}, type={type})"
.comet/scripts/comet-state set active_requirement.id REQ-{NNN}
```

---

## Step 2: 方案设计

**目标**: 产出设计方案，写入 `openspec/changes/<name>/design.md`。

### 2.1 确定起始点

根据需求当前 `status`：
- `proposed` → 从 Design 开始（完整流程）
- `designed` → 跳过 Step 2，从 Step 3 开始
- `planned` → 跳过 Step 2+3，直接进入 Step 4

### 2.2 委托 brainstorming

加载 `brainstorming` Skill，输入为：

```
你正在为以下需求设计技术方案：

[需求]
标题: {title}
类型: {type}
优先级: {priority}
描述: {description}
验收标准:
{acceptance_criteria}

[KB 上下文]
{loaded_kb_entries}

[影响分析]
{impact_analysis_summary}

[NFR 评估]
{nfr_assessment_summary}

请按 DevLoop 设计流程产出设计方案，产物写入:
  openspec/changes/<change-name>/design.md

设计方案必须覆盖:
1. 架构方案（组件、数据流、接口）
2. 技术选型（如有新增依赖需说明理由）
3. 数据模型变更（如有）
4. API 变更（含兼容性说明）
5. 非功能需求满足方案（安全/性能/可访问性等，根据 NFR 评估中的触发维度）
6. 测试策略
7. 风险与缓解措施
```

### 2.3 确认点（Level 2/3）

**Level 2/3**: Design 完成后暂停，展示设计方案摘要，等待用户确认：

> "设计方案已生成: `openspec/changes/<name>/design.md`。请审核，确认后继续创建实现计划。"

**Level 1**: 不暂停，直接进入 Step 3。

**确认选项**: `confirm` (继续) | `modify` (修改后重来) | `skip` (暂缓，标记 pending)

**门禁 G3.2** — Design 阶段质量（mixed: hard + soft）:
- **hard**: `design.md` 存在且非空
- **hard**: 必须包含「架构方案」「API 变更」「测试策略」三个章节
- **hard**: 若影响分析含 NFR 触发维度，design.md 必须包含对应的「非功能需求满足方案」章节
- **soft**: LLM 评估设计方案的合理性、完整性、与技术栈的匹配度
- **soft**: LLM 评估设计方案与影响分析的一致性（有无遗漏关键模块或过度设计）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 3 3.2 "Design complete: openspec/changes/<name>/design.md"
# Level 2/3 时：
.comet/scripts/comet-state add-pending design_ready REQ-{NNN} "Design complete, waiting confirmation"
```

---

## Step 3: 实现计划

**目标**: 产出实现计划和任务清单，写入 `openspec/changes/<name>/plan.md` + `tasks.md`。

### 3.1 委托 writing-plans

加载 `writing-plans` Skill，输入为：
- `openspec/changes/<name>/design.md`（设计方案）
- `openspec/changes/<name>/proposal.md`（需求提案）
- 需求文档中的 `acceptance_criteria`（含 NFR 补充验收项）

产物写入：
- `openspec/changes/<name>/plan.md` — 实现计划（包含 Architecture、Tech Stack、Global Constraints）
- `openspec/changes/<name>/tasks.md` — bite-sized 任务清单

### 3.2 确认点（Level 3 only）

**Level 3**: Plan 完成后暂停，展示计划摘要，等待用户确认：

> "实现计划已生成: `openspec/changes/<name>/plan.md` (N tasks)。请审核，确认后开始执行。"

**Level 1/2**: 不暂停，直接进入 Step 4。

### 3.3 构建模式判定

根据任务数量自动选择执行模式：

| 条件 | 模式 | Skill |
|------|------|-------|
| tasks > 5 或涉及模块 > 2 | 子代理驱动 | `subagent-driven-development` |
| tasks ≤ 5 且涉及模块 ≤ 2 | 内联执行 | `executing-plans` |

**工作空间隔离**: 涉及模块 > 3 或 tasks > 10 时推荐使用 `using-git-worktrees` 隔离。

**门禁 G3.3** — Plan 阶段质量（mixed: hard + soft）:
- **hard**: `plan.md` 和 `tasks.md` 存在且非空
- **hard**: `plan.md` 包含「Goal」「Architecture」「Tech Stack」「Global Constraints」章节
- **hard**: `tasks.md` 的每个 task 包含「Files」「Interfaces」「Steps」三个小节
- **hard**: `tasks.md` 中所有 task 的 checkbox 总数 >= 需求文档中 `acceptance_criteria` 的数量
- **soft**: LLM 评估 plan.md 的 task 分解粒度是否合理（每个 task 独立可测试）
- **soft**: LLM 评估 plan 与 design 的一致性（有无遗漏设计章节）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 3 3.3 "Plan complete: openspec/changes/<name>/plan.md ({N} tasks, mode={sdd|inline})"
# Level 3 时：
.comet/scripts/comet-state add-pending plan_ready REQ-{NNN} "Plan complete, waiting confirmation"
```

---

## Step 4: 代码实现

**目标**: 执行实现计划，产出代码变更和测试。

### 4.1 子代理驱动模式（大需求）

加载 `subagent-driven-development` Skill。

编排层职责（协调者）:
1. 读取 `openspec/changes/<name>/plan.md` + `tasks.md`
2. 按 tasks.md 顺序逐 task 分派 implementer subagent
3. 每个 task 完成后触发双审查（spec compliance + code quality）
4. 审查通过后调用 `comet-state complete-step build <task-id>` 标记完成
5. 审查不通过 → 分派 fix subagent → 重新审查 → 最多 3 轮
6. 全部 task 完成后触发 whole-branch review

**工作空间**: 若判定需要隔离，先加载 `using-git-worktrees` 创建隔离 worktree。

### 4.2 内联执行模式（小需求）

加载 `executing-plans` Skill。

主会话直接逐 task 执行：
1. 读取 task → 标记 in_progress → 执行步骤 → 验证 → 标记 completed
2. 每个 task 完成后 git commit（不得积攒）
3. 全部完成后触发最终验证

### 进度追踪

```bash
# 每个 task 完成后
.comet/scripts/comet-state complete-step build <task-id>
# 更新 active_requirement 进度
.comet/scripts/comet-state set active_requirement.completed_tasks <N>/<total>
```

**门禁 G3.4** — Build 阶段质量（hard）:
- 所有 task checkbox 已勾选（tasks.md 中 `[x]` 数量 = 总数）
- 所有测试通过（`pytest` / `npm test` 等退出码 0）
- 代码已提交（git log 中有对应 task 的 commit）
- 无 merge conflict
- 若使用 worktree，已合并回主分支

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 3 3.4 "Build complete: {N} tasks done, {M} commits"
```

---

## Step 5: 门禁校验

运行阶段三全部门禁：

```bash
.comet/scripts/devloop-guard check stage-3
```

必须看到 `ALL CHECKS PASSED` 或确认失败项可接受后方可继续。

**门禁清单 G3.1 - G3.4**（汇总判定）：
- [ ] G3.1: 需求可构建性通过（文档存在、验收标准非空、无 blocked 标记）
- [ ] G3.2: Design 阶段质量（hard checks 全部通过，soft checks 已评估）
- [ ] G3.3: Plan 阶段质量（hard checks 全部通过，soft checks 已评估）
- [ ] G3.4: Build 阶段质量（tasks 全部完成、测试全部通过、已提交）
- [ ] NFR 满足方案在 design.md 中有对应章节（若影响分析中有触发维度）
- [ ] 代码变更与需求文档 `acceptance_criteria` 一一对应（可追溯）

**软门禁处理**: 若 guard 输出 `SOFT_GATES_PENDING`，由编排层委托 LLM agent 逐项评估软门禁（G3.2-soft, G3.3-soft），结果写入 `.comet/gate-results/soft-gate-g3.<n>.json`。软门禁只能 WARN 不阻断。

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G3 <pass|fail> "Stage-3 gates complete"
```

---

## Step 6: Git 提交

更新需求状态并提交：

```bash
# 更新需求文档 status
# 编辑 requirements/in-progress/REQ-{NNN}-{slug}.md frontmatter:
#   status: built

git add requirements/in-progress/REQ-{NNN}-{slug}.md
git add openspec/changes/<change-name>/
git add .comet/loop-state.yaml
git commit -m "[devloop] stage:3 action:build req:REQ-{NNN}

- type: {type}
- priority: {priority}
- tasks_completed: {N}/{total}
- build_mode: {sdd|inline}
- tests_pass: {Y/N}
- nfr_dimensions_addressed: {M}

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**状态保存**:
```bash
.comet/scripts/comet-state complete-step 3 all
.comet/scripts/comet-state set active_requirement.status built
```

---

## 跳过/恢复机制

### 指定起始步骤

```bash
# 从 Design 开始（默认，status=proposed）
/devloop build REQ-010

# 跳过 Design，从 Plan 开始（status=designed）
/devloop build REQ-010 --from plan

# 跳过 Design+Plan，直接执行（status=planned）
/devloop build REQ-010 --from build
```

### 中断恢复

若在阶段三中断：
1. 运行 `comet-state check <name> build --recover`
2. 读取 `active_requirement` 和最后 checkpoint
3. 从断点步骤继续（已完成的步骤不重复执行）
4. 已生成的产物（design.md / plan.md / tasks.md）保留不覆盖

---

## 复用清单

| 调用 | 用途 |
|------|------|
| `brainstorming` | 方案设计（提供需求文档 + KB 上下文作为输入） |
| `writing-plans` | 创建实现计划（生成 plan.md + tasks.md） |
| `subagent-driven-development` | 执行实现（大需求，>5 tasks，后台子代理） |
| `executing-plans` | 执行实现（小需求，≤5 tasks，主会话） |
| `using-git-worktrees` | 隔离工作空间（可选，大需求/多模块推荐） |
| *(sdd 内部调用)* `test-driven-development` | 每个 task 的 TDD |
| *(sdd 内部调用)* `requesting-code-review` | 每个 task 的双审查 |
| `comet-state` (Phase 0) | checkpoint + 状态管理 |
| `devloop-guard` (Phase 0) | 门禁 G3.1-G3.4 |

### 编排层新建逻辑

| 逻辑 | 说明 |
|------|------|
| 需求拉取 | 扫描 `requirements/in-progress/` → 优先级排序 → 选最高分 |
| 确认点管理 | Level 2: Design 后暂停；Level 3: Design+Plan 后各暂停一次 |
| 构建模式选择 | tasks > 5 / 涉及模块 > 2 → `sdd`；否则 → `executing-plans` |
| 工作空间隔离 | 涉及模块 > 3 或 tasks > 10 → 推荐 `using-git-worktrees` |
| 产物路径映射 | design.md / plan.md / tasks.md 统一写入 `openspec/changes/<name>/` |
| 进度追踪 | 每个 task 完成后 `comet-state complete-step`；更新 completed_tasks 计数 |
| 跳过机制 | `--from plan` / `--from build` 跳过已完成步骤，复用已有产物 |

---

## 确认级别影响

| 维度 | Level 1 (低保障) | Level 2 (标准保障) | Level 3 (高保障) |
|------|-----------------|-------------------|-----------------|
| 触发条件 | `bugfix` + `low` + ≤1 模块 | 默认 | `high` 或 `feature` 或 ≥3 模块 |
| Design 后确认 | ❌ 不暂停 | ✅ 暂停 | ✅ 暂停 |
| Plan 后确认 | ❌ 不暂停 | ❌ 自动继续 | ✅ 暂停 |
| 构建模式 | `executing-plans`（默认） | `executing-plans`（默认） | `sdd`（若 tasks > 3） |
| Worktree 隔离 | ❌ 不推荐 | 可选 | ✅ 推荐（多模块时） |
| 审查强度 | 单审查者 | 双审查者 | 双审查者 + 安全审查 |

---

## 防御原则

1. **确认点不可跳过**: Level 2/3 的 Design/Plan 确认暂停点必须等待用户明确选择，不得自动填充默认值
2. **产物路径统一**: design.md / plan.md / tasks.md 统一写入 `openspec/changes/<name>/`，不散落各处
3. **进度可恢复**: 每个 Step 结束保存 checkpoint，中断后可从断点继续，不重复生成已有产物
4. **门禁不可绕过**: 所有 G3.x 门禁（含软门禁 LLM 评估）必须在提交前通过
5. **NFR 不能丢失**: design.md 必须覆盖需求文档中 NFR 评估触发的所有维度；若 design 遗漏任何触发维度，G3.2 hard check 阻止进入 Plan
6. **禁止在无设计的情况下直接写代码**: 必须先产出 design.md + plan.md，再进入 Step 4 执行。即使 Level 1 不暂停确认，design 和 plan 也仍然生成
7. **增量提交**: 每个 task 完成后立即 git commit，不得积攒多个 task 的变更
