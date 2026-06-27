---
name: devloop-loop
description: "DevLoop 阶段五：闭环调度 — 扫描队列→优先级排序→状态路由→触发下一阶段。"
---

# DevLoop Loop — 阶段五：闭环调度

扫描需求队列，按优先级选取下一个需求，根据状态路由到对应的处理阶段，形成端到端闭环。

## 触发条件

1. 阶段四完成后自动触发（需求 completed → 找下一个）
2. 用户手动触发：`/devloop loop`
3. 定时触发（通过 cron / schedule）
4. 会话启动时检测到有待处理需求

## 流程总览

```
Step 1: 队列扫描        → (编排层) requirements/in-progress/        → 活跃需求列表
Step 2: 优先级排序       → (编排层) priority_score + dependency_score → 选取最高分
Step 3: 状态路由         → (编排层) status → 对应 Skill              → 触发下一阶段
Step 4: 门禁校验         → devloop-guard check stage-5              → G5.1
Step 5: 队列空处理       → (编排层) 状态摘要 + 等待提示              → 等待新需求或结束
```

---

## Step 1: 队列扫描

**目标**: 扫描 `requirements/in-progress/` 目录，找出所有待处理的需求。

### 1.1 扫描逻辑

```bash
# 列出所有进行中的需求
ls requirements/in-progress/REQ-*.md 2>/dev/null
```

对每个需求文件，读取 frontmatter 提取：

```yaml
id: REQ-{NNN}
title: {标题}
type: feature | bugfix | enhancement | refactor | chore
priority: high | medium | low
status: proposed | designed | planned | built | blocked
confirmation_level: 1 | 2 | 3
dependencies: []      # 依赖的其他 REQ ID
```

### 1.2 过滤条件

排除以下状态的需求：
- `status: completed` — 已归档到 `requirements/completed/`
- `status: blocked` — 被阻塞（依赖未满足或等待外部输入）
- `status: rejected` — 已驳回

**保留的需求状态**:

| status | 含义 | 路由目标 |
|--------|------|---------|
| `proposed` | 需求已创建，尚未设计 | `devloop-intake`（从完善开始） |
| `designed` | 设计已完成，尚未制定计划 | `devloop-build --from plan` |
| `planned` | 计划已完成，尚未执行 | `devloop-build --from build` |
| `built` | 代码已生成，等待验证 | `devloop-verify` |

### 1.3 队列容量检查

若 `requirements/in-progress/` 中无匹配的需求文件：
→ 跳转 Step 5（队列空处理）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 5 5.1 "Queue scan: {N} active requirements"
```

---

## Step 2: 优先级排序

**目标**: 从活跃需求中选取最高优先级的一个。

### 2.1 评分公式

```
priority_score = {
  high:   3
  medium: 2
  low:    1
}[priority]

dependency_score = |completed_dependencies| / |total_dependencies|
# 无依赖时 dependency_score = 1.0
# 有依赖时: 已完成的依赖数 / 总依赖数

composite_score = priority_score * 0.7 + dependency_score * 0.3
```

### 2.2 排序规则

1. 按 `composite_score` 降序排列
2. 同分时按 `status` 优先级: `built > planned > designed > proposed`（越接近完成越优先）
3. 仍同分时按 `confirmation_level` 升序（低保障快速通道优先）
4. 仍同分时按 `REQ-{NNN}` ID 升序（先进先出）

### 2.3 选取结果

```bash
# 将选中的需求设为 active
.comet/scripts/comet-state set active_requirement.id REQ-{NNN}
.comet/scripts/comet-state set active_requirement.title "{标题}"
.comet/scripts/comet-state set active_requirement.priority "{priority}"
.comet/scripts/comet-state set active_requirement.status "{status}"
```

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 5 5.2 "Selected: REQ-{NNN} (score={composite_score}, status={status})"
```

---

## Step 3: 状态路由

**目标**: 根据选中需求的 `status` 路由到对应的 DevLoop 阶段。

### 3.1 路由表

| status | 路由目标 | 加载的 Skill | 说明 |
|--------|---------|-------------|------|
| `proposed` | 阶段二：需求完善 | `devloop-intake` | 从影响分析 / NFR 评估开始继续完善需求 |
| `designed` | 阶段三：从 Plan 开始 | `devloop-build --from plan` | 跳过 Design，直接创建 Plan |
| `planned` | 阶段三：从 Build 开始 | `devloop-build --from build` | 跳过 Design+Plan，直接执行 |
| `built` | 阶段四：验证 | `devloop-verify` | 验证→审查→修复→收尾→完成 |

### 3.2 路由执行

```
if status == "proposed":
    1. 加载 devloop-intake Skill
    2. 传入 REQ-{NNN} 和目标
    3. devloop-intake 从 Step 3（影响分析）或 Step 5（需求文档完善）继续
    4. 完成后 status 变为 designed → 再次进入 loop

if status == "designed":
    1. 加载 devloop-build Skill（--from plan）
    2. 跳过 Design 步骤，直接从 Plan 开始
    3. 完成后 status 变为 built → 再次进入 loop

if status == "planned":
    1. 加载 devloop-build Skill（--from build）
    2. 跳过 Design + Plan，直接执行 Build
    3. 完成后 status 变为 built → 再次进入 loop

if status == "built":
    1. 加载 devloop-verify Skill
    2. 执行验证→审查→修复→收尾
    3. 完成后 status 变为 completed → git mv 到 completed/
        → 再次进入 loop（找下一个需求）
```

### 3.3 路由后行为

被调用的阶段 Skill 执行完毕后，控制权返回 `devloop-loop`：
- **需求未完成**（status 推进但未到 completed）→ 重新扫描队列（可能同一个需求推进了状态，也可能有更高优先级的需求插入）
- **需求已完成**（status = completed）→ `active_requirement` 清空，重新扫描队列选下一个
- **需求被阻塞**（status = blocked）→ 跳过该需求，重新排序选下一个

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 5 5.3 "Routing: REQ-{NNN} (status={status}) → {target_skill}"
```

---

## Step 4: 门禁校验

运行阶段五门禁：

```bash
.comet/scripts/devloop-guard check stage-5
```

**门禁 G5.1** — 闭环状态一致性（hard）:
- `active_requirement` 非空时，其 `status` 与对应 `requirements/in-progress/REQ-*.md` frontmatter 一致
- `active_requirement` 为空时，`requirements/in-progress/` 中无 `status ∈ [proposed, designed, planned, built]` 的需求
- `requirements/completed/` 中的需求 `status` 均为 `completed`
- `loop-state.yaml` 中的 `current_stage` 与实际 loaded Skill 一致
- 无「僵尸需求」：`status = proposed` 超过 7 天无更新 → 标记为 stale

**失败动作**: `warn_and_continue`（状态不一致时告警但继续，不阻塞 loop）

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G5.1 <pass|warn> "Stage-5 gates complete"
```

---

## Step 5: 队列空处理

**目标**: 当无待处理需求时，输出状态摘要并给出明确提示。

### 5.1 状态摘要

```
✅ DevLoop 闭环状态

📊 需求总览:
  已完成: {completed_count} 个
  进行中: {in_progress_count} 个
  已阻塞: {blocked_count} 个

📋 最近完成:
  - REQ-{NNN}: {title} (completed at {timestamp})
  ...

⏳ 等待新需求输入。
  提交新需求: /devloop intake <需求描述>
  手动触发 loop: /devloop loop
```

### 5.2 等待模式

- **手动模式**（默认）: 输出摘要后结束，等待用户输入新需求
- **守护模式**（`/devloop loop --watch`）: 定期扫描队列（默认每 5 分钟），检测到新需求自动触发处理
- **定时模式**（通过 cron）: `*/5 * * * * /devloop loop`

### 5.3 已阻塞需求处理

若存在 `status: blocked` 的需求：

```
⚠️ {blocked_count} 个需求被阻塞:

  - REQ-{NNN}: {title} — 阻塞原因: {blocked_reason}
    解除条件: {unblock_condition}

  使用 /devloop resume REQ-{NNN} 在阻塞解除后恢复处理。
```

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 5 5.5 "Queue empty: {completed_count} completed, {blocked_count} blocked"
```

---

## 中断恢复

若在阶段五中断（如被调用的阶段 Skill 执行中会话超时）：

1. 会话重启后加载 `devloop-resume` Skill（优先级高于 `devloop-loop`）
2. `devloop-resume` 从中断的阶段恢复执行
3. 该阶段完成后，`devloop-loop` 重新扫描队列继续

---

## 复用清单

| 调用 | 用途 |
|------|------|
| `devloop-intake` (T2.1) | 路由: `proposed` → 阶段二（需求完善） |
| `devloop-build` (T3.1) | 路由: `designed`/`planned` → 阶段三（代码生成） |
| `devloop-verify` (T4.1) | 路由: `built` → 阶段四（验证与闭环） |
| `devloop-resume` (T0.4) | 中断恢复（优先级高于 loop） |
| `comet-state` (Phase 0) | 状态管理 + checkpoint |
| `devloop-guard` (Phase 0) | 门禁 G5.1 |

### 编排层新建逻辑

| 逻辑 | 说明 |
|------|------|
| 队列扫描 | 扫描 `requirements/in-progress/`，过滤 `completed`/`blocked`/`rejected` |
| 优先级排序 | `composite_score = priority_score * 0.7 + dependency_score * 0.3`；同分时按 status → confirmation_level → ID 排序 |
| 状态路由 | `proposed → devloop-intake`，`designed → devloop-build --from plan`，`planned → devloop-build --from build`，`built → devloop-verify` |
| 重新入队 | 阶段完成后需求未到 completed → 重新扫描（允许高优先级需求抢占） |
| 队列空处理 | 输出状态摘要 + 等待提示；支持 `--watch` 守护模式 |
| 僵尸检测 | `proposed` 超过 7 天 → stale 标记 |
| 阻塞需求展示 | 列出 `blocked` 需求及解除条件 |

---

## 防御原则

1. **状态一致性**: `active_requirement` 必须与 `requirements/in-progress/` 中的 frontmatter 一致。每次 loop 开始前验证 G5.1
2. **优先级不可绕过**: 始终选取 `composite_score` 最高的需求，不得手工挑选（除非用户显式指定 `REQ-ID`）
3. **路由准确性**: 根据 `status` 精确路由到正确阶段。`proposed` 不能直接跳到 Build，`built` 不能跳过 Verify
4. **循环有界**: 每个需求在单次 loop 中只推进一个阶段。阶段完成后重新入队，允许高优先级需求插入
5. **空队列明确**: 队列为空时明确告知用户状态，不隐式等待。守护模式下明确间隔
6. **阻塞可见**: `blocked` 需求不被静默跳过，每次 loop 展示阻塞列表和解除条件
7. **中断安全**: loop 本身不持有状态——每个阶段 Skill 独立管理 checkpoint。中断后由 `devloop-resume` 恢复，然后 loop 继续
