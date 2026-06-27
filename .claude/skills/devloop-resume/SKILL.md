---
name: devloop-resume
description: "DevLoop 中断恢复 — 会话重启时读取 loop-state.yaml 回当前进度，处理待确认项，从 checkpoint 继续。由 comet-state 脚本驱动恢复逻辑。"
---

# DevLoop Resume — 中断恢复

DevLoop 闭环可能因会话超时、上下文压缩、API 错误或人工确认等待而中断。
此 Skill 负责从中断点安全恢复，尽可能少地丢失上下文。

## 触发条件

以下情况必须加载此 Skill：
1. 会话重启后检测到 `.comet/loop-state.yaml` 存在活跃流程
2. 上下文压缩后需要恢复之前的工作状态
3. 用户手动运行 `/devloop resume`

## 恢复流程

### Step 1: 运行恢复检查

```bash
# 读取当前状态
.comet/scripts/comet-state check <change-name> <phase> --recover
```

此命令会输出：
- 当前阶段和步骤
- 待确认项数量
- 失败的门禁
- 推荐的恢复动作

### Step 2: 快速上下文重建

按以下优先级读取上下文文件：

1. **当前 change 的 proposal.md** — 知道在做什么
   ```
   openspec/changes/<change-name>/proposal.md
   ```

2. **loop-state.yaml** — 知道做到哪了
   ```
   .comet/loop-state.yaml → current_stage, current_step, checkpoint
   ```

3. **KB 相关模块** — 知道代码上下文
   ```
   knowledge-base/modules/<module>/overview.md
   ```

4. **最近 memory 文件** — 知道决策历史
   ```
   ~/.claude/projects/<project>/memory/*.md
   ```

### Step 3: 处理待确认项

如果 `comet-state check --recover` 报告 `pending_count > 0`：

```bash
.comet/scripts/comet-state list-pending
```

对每个待确认项：
1. 展示确认项的上下文（从 loop-state.yaml 的 pending_confirmations 读取）
2. 列出关键决策点和风险
3. 提供选项：✅ 确认 / ✏️ 修改 / ⏭️ 跳过 / ❌ 放弃
4. 记录确认决策

### Step 4: 根据恢复动作继续

| Recovery Action | 行为 |
|----------------|------|
| `resume_from_step` | 从 checkpoint 的 stage.step 继续执行 |
| `fresh_start` | 状态文件不存在，初始化新流程 |
| `resume_from_pending` | 优先处理待确认项 |
| `gate_failures_present` | 检查失败门禁，决定重试或跳过 |

### Step 5: 控制权交回

恢复完成后：
- 将当前状态摘要输出给用户
- 从中断点继续 DevLoop 流程
- 调用对应阶段的 Skill（devloop-reverse / devloop-intake / devloop-build / devloop-verify / devloop-loop）

## 中断场景与恢复策略

| 场景 | 恢复动作 |
|------|---------|
| 会话超时 | 读取 checkpoint → 展示进度 → 从当前步骤继续 |
| 上下文压缩 | `comet-state check --recover` → 重建上下文 → 继续 |
| 人工确认等待 | 展示待确认项 → 请求确认 → 继续 |
| 门禁失败 | 展示失败列表 → 决定修复/跳过 → 重试门禁 |
| API 错误 | 指数退避重试（最多 3 次） |

## 状态文件结构

```yaml
# .comet/loop-state.yaml
devloop:
  version: "1.0"
  change_name: "<name>"
  current_stage: 3
  current_step: "3.4"
  active_requirement:
    id: REQ-010
    title: "..."
  checkpoint:
    stage: 3
    step: "3.4"
    last_action: "plan.md created, waiting for user confirmation"
    timestamp: "2026-06-27T10:30:00Z"
    git_commit: "abc1234"
  pending_confirmations:
    - type: plan_ready
      req: REQ-010
      message: "Implementation plan is ready for review"
  errors: []
```

## 复用

此 Skill 依赖以下基础设施（Phase 0 产物）：
- `comet-state` 脚本 — 状态读取、checkpoint 恢复
- `devloop-guard` 脚本 — 门禁状态检查
- `loop-state.yaml` — 运行时状态文件
