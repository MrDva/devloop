---
module: devloop-skills
source: auto-generated
confidence: 0.80
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - .claude/skills/
---

# devloop-skills -- 业务逻辑

## 核心流程

### DevLoop 五阶段闭环
- **触发条件**: 新项目启动 / 用户发出 `/devloop reverse` 命令 / KB 漂移检测触发
- **前置条件**: 项目代码已就位，codegraph 索引可用
- **流程步骤**:
  1. **Reverse（阶段一，逆向分析）**: 扫描代码 -> 并行生成 KB 条目 -> 汇总整合 -> 反推基线需求 -> 门禁校验 -> Git 提交
  2. **Intake（阶段二，需求确认）**: 评估 baseline requirements + 新需求 -> 确定需求级别（Level 1/2/3） -> 计算优先级 -> 生成 plan.md + tasks.md -> 门禁校验
  3. **Build（阶段三，实现构建）**: 选择 build mode -> 调度实现 agent -> 执行任务 -> 审查 -> 修复（最多 3 轮） -> 门禁校验 -> Git 提交
  4. **Verify（阶段四，验证）**: 运行验证 -> 执行 verification-before-completion -> 门禁校验
  5. **Loop（阶段五，循环决策）**: 检查是否有未覆盖需求/新变更需求 -> 回到阶段一或二，或结束流程
- **后置条件**: KB 条目创建/更新，git commit 已提交
- **异常处理**:
  - 会话中断 -> devloop-resume 自动/手动恢复
  - 门禁失败 -> 展示失败项由用户决定重试或跳过
  - 修复循环超 3 次 -> 暂停等待用户决策

### 逆向分析流程（devloop-reverse）
- **触发条件**: `/devloop reverse` / 新项目启动 / 漂移检测
- **前置条件**: codegraph 索引可用
- **流程步骤**:
  1. 代码扫描（codegraph_explore MCP）-> `.manifest.yaml`
  2. 并行模块分析（dispatching-parallel-agents）-> 每个模块 5 个 KB 文件
  3. KB 汇总整合（LLM）-> architecture/ + apis/ + data-models/
  4. 需求反向推断（brainstorming）-> requirements/baseline/REQ-*.md
  5. 门禁校验（devloop-guard）-> G1.1-G1.5
  6. Git 提交（git commit）
- **后置条件**: KB 目录完整，基线需求文档已创建
- **异常处理**:
  - G1.1 模块覆盖率不达标 -> 提示用户补充手动指定目录
  - G1.2 KB 条目缺失 -> 标记缺失模块后重分析
  - G1.5 总门禁软门禁 -> LLM agent 逐项评估并写入 `.comet/gate-results/`

### 中断恢复流程（devloop-resume）
- **触发条件**: 会话重启检测到 `.comet/loop-state.yaml` / 手动 `/devloop resume`
- **前置条件**: loop-state.yaml 存在
- **流程步骤**:
  1. 运行恢复检查（`comet-state check --recover`）
  2. 快速上下文重建（读 proposal.md, loop-state.yaml, KB overview.md, memory 文件）
  3. 处理待确认项（显示、确认、修改、跳过或放弃）
  4. 按 Recovery Action 继续（resume_from_step / fresh_start / resume_from_pending / gate_failures_present）
  5. 调用对应阶段 Skill 继续执行
- **后置条件**: 流程从中断点继续
- **异常处理**:
  - 上下文压缩 -> 通过 comet-state 重建上下文
  - API 错误 -> 指数退避重试（最多 3 次）

## 业务规则

### 需求确认级别判定（intake 阶段）
- **Level 1**: 已有基线需求覆盖，无需额外分析 -- 直接进入 Build
- **Level 2**: 需要 brainstorming 和 design doc -- 补充设计后进入 Build
- **Level 3**: 影响多模块或架构变更 -- 需要暂停等待用户拆分和确认

### 任务优先级排序算法
- `priority_score = urgency_weight * 0.4 + impact_weight * 0.3 + dependency_score * 0.3`
- `dependency_score` 根据依赖图计算：被依赖越多优先级越高
- 排序结果写入 tasks.md，高优先级先执行

### Build 模式选择
- **subagent-driven-development（SDD 模式）**: 当任务大多独立且子 agent 可用时。每个 task 一个独立子 agent + 任务审查（spec compliance + code quality），最终全局审查
- **executing-plans（EP 模式）**: 无子 agent 支持时的降级方案。单会话顺序执行 plan，完成后调用 finishing-a-development-branch

### 修复循环规则
- 每轮修复前必须加载 `systematic-debugging` 找到根因
- 最大连续修复轮次：3 次
- 超限后必须暂停，等待用户选择：接受偏差 / 调整方案 / 继续修复
- 修复过程中禁止跳过根因分析直接提修复方案

### 门禁规则（Reverse 阶段）
- **G1.1 模块覆盖率**: modules_analyzed / total_modules >= 0.8
- **G1.2 KB 条目完整性**: 每个模块必须有 overview.md + api.md
- **G1.3 KB 一致性**: `module-name` wiki link 引用目标存在且语义匹配
- **G1.4 需求覆盖率**: covered_modules / total_modules >= 0.9；low confidence <= 20%
- **G1.5 总门禁**: 所有前序门禁通过 + git add 完成 + KB 大小 > 0

### 基线需求约束
- `source: auto-inferred` 和 `usage_restriction: reference_only` 必须保留
- 基线需求仅供阶段二影响分析参考，不得作为阶段三设计方案的强制约束
- 新需求与基线需求冲突时，以新需求为准

### 增量模式规则（devloop-reverse --scope）
- Step 1 跳过（复用已有 manifest）
- Step 2 仅分析指定模块
- Step 3 增量更新汇总文档
- Step 4 仅更新涉及模块的基线需求（已有需求标记 updated）
- Step 5/6 正常执行

## 副作用

- **KB 目录修改**: 每个 devloop-reverse 执行创建/更新 `knowledge-base/` 下的文件
- **Git 状态变更**: 每阶段提交 git commit，分支名为 `kb/update-<date>`（阶段一）或按阶段命名
- **state 文件更新**: `.comet/loop-state.yaml` 和 `.comet/state/` 下的 checkpoint 文件被频繁读写
- **门禁文件创建**: `.comet/gate-results/` 下的 JSON 文件在软门禁评估时生成
- **OpenSpec 变更目录**: openspec/changes/ 下的目录结构由各 openspec-* Skill 创建、修改或归档
- **文件系统副作用**: `comet-state checkpoint` 和 `devloop-guard` 脚本每次执行都会产生日志输出

---
