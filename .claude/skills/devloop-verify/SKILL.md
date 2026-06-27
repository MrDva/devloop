---
name: devloop-verify
description: "DevLoop 阶段四：验证与闭环 — 验证→审查→修复循环→收尾→更新需求状态→提交。"
---

# DevLoop Verify — 阶段四：验证与闭环

对阶段三产出的代码变更执行验证、审查和修复循环，通过后收尾合并并将需求标记为已完成。

## 触发条件

1. 阶段三 Build 完成后自动触发（由 `devloop-build` 或 `devloop-loop` 调用）
2. 用户手动触发：`/devloop verify REQ-<NNN>`
3. `devloop-loop` 检测到 `status: built` 的需求

## 流程总览

```
Step 1: 加载构建上下文     → (编排层 LLM)                              → 确定目标需求 + 代码变更范围
Step 2: 验证执行            → verification-before-completion            → 测试通过证据
Step 3: 代码审查            → code-review (内置)                        → 审查报告
Step 4: 修复循环             → systematic-debugging                      → 修复 ≤3 次自动重试，>3 次暂停
Step 5: 门禁校验             → devloop-guard check stage-4              → G4.1 - G4.4
Step 6: 收尾合并             → finishing-a-development-branch           → 合并/PR/保留/丢弃
Step 7: 需求状态更新         → (编排层) status→completed + git mv      → requirements/completed/
Step 8: Git 提交             → git commit                                → [devloop] stage:4 action:verify
```

每个 Step 结束后调用 `comet-state checkpoint` 保存断点，支持中断恢复。

---

## Step 1: 加载构建上下文

**目标**: 确定本次验证的目标需求及其构建产物。

### 1.1 确定目标需求

从 `loop-state.yaml` 的 `active_requirement` 或用户指定参数获取目标 `REQ-<NNN>`。

加载需求文档 `requirements/in-progress/REQ-<NNN>-<slug>.md`，提取：
- `id`, `title`, `type`, `priority`, `status`
- `acceptance_criteria`（含 NFR 补充验收项）
- `affected_modules`, `api_changes`, `data_model_changes`

### 1.2 确定代码变更范围

```bash
# 查找阶段三的提交
git log --oneline --grep="\[devloop\] stage:3.*req:REQ-<NNN>" -5
```

确认：
- `openspec/changes/<name>/plan.md` + `tasks.md` 存在
- 所有 task checkbox 已勾选（G3.4 应已通过）
- 测试命令已知（从 `plan.md` 或项目约定中提取）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 4 4.1 "Verify context loaded: REQ-{NNN} ({N} acceptance criteria)"
```

---

## Step 2: 验证执行

**目标**: 运行验证命令，获取测试通过的实际证据。

### 2.1 委托 verification-before-completion

加载 `verification-before-completion` Skill，遵循其 Iron Law：

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

**执行步骤**（由 verification-before-completion Skill 驱动）:
1. **IDENTIFY**: 确定证明「代码正确」的验证命令（从 plan.md / 项目约定获取）
2. **RUN**: 执行完整验证命令（全新运行，不依赖缓存）
3. **READ**: 读取完整输出，检查退出码，统计失败数
4. **VERIFY**: 对照 `acceptance_criteria` 逐条确认
5. **ONLY THEN**: 产出验证报告

### 2.2 验证策略

| 验证类型 | 命令来源 | 通过标准 |
|---------|---------|---------|
| 单元测试 | 项目测试框架（pytest / npm test / go test） | 退出码 0，0 failures |
| 需求验收 | 逐条对照 `acceptance_criteria`（含 NFR 补充项） | 全部满足 |
| 构建验证 | 项目构建命令（npm build / cargo build） | 退出码 0 |
| NFR 验证 | NFR 评估中触发的维度 | 对应测试/检查通过 |

**聚焦输出正确性**: 测试通过 = 正确。不要求环境一致（如 CI 环境 vs 本地环境差异不阻塞）。

### 2.3 验证报告

产出结构化验证报告：

```markdown
## 验证报告 — REQ-{NNN}

**验证时间**: {ISO-8601}
**验证命令**: {command}

### 测试结果
- 总测试数: {N}
- 通过: {P}
- 失败: {F}
- 跳过: {S}

### 验收标准逐条确认
- [x] AC-001: {描述} — ✅ 通过
- [x] AC-002: {描述} — ✅ 通过
- [ ] AC-003: {描述} — ❌ 未满足（进入修复循环）

### NFR 验收项
- [x] NFR-Auth: {描述} — ✅ 通过
...
```

**门禁 G4.1** — 自动化验证（hard）:
- 测试命令退出码为 0
- 0 failures
- 所有 `acceptance_criteria` 对应测试通过
- 所有 NFR 触发维度的验收项通过

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 4 4.2 "Verification complete: {P}/{N} pass, {F} failures"
# 若全部通过
.comet/scripts/comet-state gate-result G4.1 pass "All tests pass, all AC met"
# 若有失败
.comet/scripts/comet-state gate-result G4.1 fail "{F} failures, entering fix loop"
```

---

## Step 3: 代码审查

**目标**: 对代码变更进行审查，识别正确性问题和代码质量问题。

### 3.1 委托 code-review

调用内置 `code-review` Skill（通过 `/code-review` 或直接调用）。

审查范围：阶段三产生的所有代码变更（`git diff` 从阶段三第一个 commit 到 HEAD）。

**审查维度**:
| 维度 | 检查内容 |
|------|---------|
| 正确性 | 逻辑错误、边界条件、空值处理、竞态条件 |
| 安全性 | 注入风险、权限校验、敏感数据处理 |
| 性能 | 不必要的循环、N+1 查询、内存泄漏 |
| 可维护性 | 命名、注释、代码重复、函数长度 |

### 3.2 审查结果处理

审查产出的 finding 按严重程度处理：

| 严重度 | 处理方式 |
|--------|---------|
| CRITICAL | 必须在修复循环中解决 |
| WARNING | 建议修复，记录到修复循环 |
| INFO | 记录但不阻塞通过 |

**门禁 G4.2** — 代码审查（mixed: hard + soft）:
- **hard**: 审查已执行，审查报告存在
- **hard**: 无 CRITICAL 级别 finding 未解决
- **soft**: LLM 评估审查覆盖的全面性（是否遗漏关键检查维度）
- **soft**: LLM 评估 CRITICAL/WARNING finding 的修复质量

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 4 4.3 "Code review complete: {C} critical, {W} warnings, {I} info"
.comet/scripts/comet-state gate-result G4.2 <pass|fail> "Code review: {C} critical findings"
```

---

## Step 4: 修复循环

**目标**: 对验证失败和审查发现的问题进行修复，≤3 次自动重试，>3 次暂停等人工决策。

### 4.1 委托 systematic-debugging

对每个失败项，加载 `systematic-debugging` Skill：

```
根因 → 假设 → 验证 → 修复
```

**严禁**在根因未定位前提出源码修复。

### 4.2 修复循环逻辑

```
attempt = 0
failures = [G4.1 failures] + [G4.2 CRITICAL findings]

while failures not empty AND attempt < 3:
    attempt += 1
    for each failure in failures:
        1. systematic-debugging → 定位根因
        2. 提出修复方案
        3. 应用修复
        4. 重新运行验证（Step 2）
        5. 若通过 → 标记已修复
        6. 若仍失败 → 进入下一轮

if failures not empty AND attempt >= 3:
    暂停，展示未解决的失败项，等待用户决策：
    - "retry": 再试 3 次
    - "accept": 接受偏差（记录到 known_gaps）
    - "abort": 放弃此需求
```

### 4.3 修复记录

每轮修复后记录：

```bash
.comet/scripts/comet-state checkpoint 4 4.4 "Fix round {attempt}: {fixed}/{total} resolved"
```

**门禁 G4.3** — 修复循环限制（hard）:
- 修复轮次 ≤ 3
- 超过 3 次后必须用户确认方可继续
- 若用户选择 `accept`，偏差记录到需求文档 `known_gaps` 字段

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G4.3 <pass|fail> "Fix loop: {N} rounds, {F} remaining failures"
```

---

## Step 5: 门禁校验

运行阶段四全部门禁：

```bash
.comet/scripts/devloop-guard check stage-4
```

必须看到 `ALL CHECKS PASSED` 或确认失败项可接受后方可继续。

**门禁清单 G4.1 - G4.4**（汇总判定）：
- [ ] G4.1: 自动化验证通过（测试 0 failures，所有 AC 满足）
- [ ] G4.2: 代码审查通过（无 CRITICAL 未解决，soft checks 已评估）
- [ ] G4.3: 修复循环合规（≤3 轮或用户已确认接受偏差）
- [ ] G4.4: 阶段四总门禁（汇总 G4.1-G4.3 全部通过）

**软门禁处理**: 若 guard 输出 `SOFT_GATES_PENDING`，由编排层委托 LLM agent 逐项评估软门禁（G4.2-soft），结果写入 `.comet/gate-results/soft-gate-g4.2.json`。软门禁只能 WARN 不阻断。

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G4.4 <pass|fail> "Stage-4 gates complete"
```

---

## Step 6: 收尾合并

**目标**: 将验证通过的代码变更合并回主分支或创建 PR。

### 6.1 委托 finishing-a-development-branch

加载 `finishing-a-development-branch` Skill。

该 Skill 会：
1. 验证测试（再次确认）
2. 检测环境（普通仓库 / worktree / detached HEAD）
3. 展示 4 个选项（或 3 个，detached HEAD）
4. 执行用户选择（合并 / PR / 保留 / 丢弃）
5. 清理 workspace（如适用）

**编排层职责**: 将控制权交给 `finishing-a-development-branch`，由用户选择收尾方式。编排层不自动做合并决策。

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 4 4.6 "Branch finishing: {option} selected"
```

---

## Step 7: 需求状态更新

**目标**: 将需求从 `in-progress` 归档到 `completed`，更新状态。

### 7.1 更新需求文档

编辑 `requirements/in-progress/REQ-{NNN}-<slug>.md` frontmatter：

```yaml
status: completed                    # proposed/designed/planned/built → completed
completed_at: {ISO-8601 timestamp}
verification_summary:                # Step 2 产出
  tests_total: {N}
  tests_passed: {P}
  tests_failed: 0
review_summary:                      # Step 3 产出
  critical_findings: 0
  warnings_addressed: {W}
known_gaps: []                       # Step 4 接受的偏差（如有）
```

### 7.2 移动到 completed 目录

```bash
git mv requirements/in-progress/REQ-{NNN}-<slug>.md requirements/completed/REQ-{NNN}-<slug>.md
```

### 7.3 更新循环状态

```bash
.comet/scripts/comet-state set active_requirement.status completed
.comet/scripts/comet-state set active_requirement.completed_at "{ISO-8601}"
```

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 4 4.7 "Requirement completed: REQ-{NNN} moved to completed/"
```

---

## Step 8: Git 提交

```bash
git add requirements/completed/REQ-{NNN}-<slug>.md
git add requirements/in-progress/   # 反映 git mv 变更
git add openspec/changes/<change-name>/
git add .comet/loop-state.yaml
git commit -m "[devloop] stage:4 action:verify req:REQ-{NNN}

- status: completed
- tests: {P}/{N} pass
- review: 0 critical findings
- fix_rounds: {N}
- branch_action: {merge|pr|keep}
- completed_at: {ISO-8601}

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**状态保存**:
```bash
.comet/scripts/comet-state complete-step 4 all
```

---

## 修复循环速查

| 轮次 | 动作 | 决策 |
|------|------|------|
| 第 1 轮 | 自动：定位根因 → 修复 → 重测 | 自动继续 |
| 第 2 轮 | 自动：定位根因 → 修复 → 重测 | 自动继续 |
| 第 3 轮 | 自动：定位根因 → 修复 → 重测 | 自动继续 |
| 第 4 轮 | 暂停：展示未解决问题 | 用户选择: retry / accept / abort |

**accept 的偏差记录格式**:
```yaml
known_gaps:
  - id: GAP-001
    description: "{描述}"
    severity: minor | acceptable
    reason: "{接受原因}"
    accepted_by: user
    accepted_at: {ISO-8601}
```

---

## 复用清单

| 调用 | 用途 |
|------|------|
| `verification-before-completion` | 验证纪律（必须看到测试通过的实际输出） |
| `code-review` (内置) | 代码审查（正确性/安全性/性能/可维护性） |
| `systematic-debugging` | 修复失败（根因→假设→验证→修复） |
| `finishing-a-development-branch` | 收尾合并（合并/PR/保留/丢弃 + workspace 清理） |
| `comet-state` (Phase 0) | checkpoint + 状态管理 |
| `devloop-guard` (Phase 0) | 门禁 G4.1-G4.4 |

### 编排层新建逻辑

| 逻辑 | 说明 |
|------|------|
| 修复循环 | 失败 ≤3 次自动重试（systematic-debugging），>3 次暂停等人工决策（retry/accept/abort） |
| 验证策略 | 聚焦输出正确性（测试通过=正确），不要求环境一致；对照 acceptance_criteria 逐条确认 |
| NFR 验证 | 对照 NFR 触发维度的补充验收标准逐条验证 |
| 偏差记录 | accept 时记录 known_gaps 到需求文档，含原因和时间戳 |
| 需求状态更新 | `status: completed` + `git mv in-progress/ → completed/` + 归档时间戳 |
| 收尾委托 | 将合并决策权交给 `finishing-a-development-branch`，编排层不自动决定合并方式 |

---

## 防御原则

1. **证据先于声明**: 必须运行验证命令并看到实际通过输出，不得凭记忆或推测声称通过。遵循 `verification-before-completion` 的 Iron Law
2. **修复必须有根因**: 任何修复必须先通过 `systematic-debugging` 定位根因。禁止在未理解根因的情况下改代码
3. **修复循环上限**: 最多 3 轮自动重试。第 4 次失败必须暂停等待用户决策，不得无限循环
4. **偏差可追溯**: 接受的偏差必须记录到需求文档 `known_gaps`，含原因、严重度和接受时间
5. **收尾需确认**: 合并/PR/保留/丢弃的选择权交给用户（通过 `finishing-a-development-branch`），编排层不自动做合并决策
6. **状态原子更新**: 需求文档 `status` 更新 + `git mv` + `loop-state.yaml` 更新必须在同一步完成，避免状态不一致
7. **增量提交**: 验证通过后立即 git commit，不积攒多个需求的验证结果
