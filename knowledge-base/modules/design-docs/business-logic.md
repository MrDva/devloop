---
module: design-docs
source: auto-generated
confidence: 0.80
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - design/00-README.md
  - design/01-overview.md
  - design/02-detailed-design.md
  - design/03-flowcharts.md
  - design/05-implementation-tasks.md
---

# design-docs — 业务逻辑

## 1. 五阶段闭环流程规则

### 1.1 阶段一：逆向分析 (Reverse Engineering)

**触发条件**: 系统初始化 / 代码大版本变更后 / 手动触发
**前置条件**: 项目已索引（.codegraph/ 目录存在）
**输入**: 现有代码仓库
**输出**: 结构化知识库 + 基线需求文档

**流程步骤**:
1. 代码扫描 → codegraph_explore (MCP) 识别模块清单
2. 并行模块深度分析 → dispatching-parallel-agents，每个 agent 内部用 codegraph_node
3. KB 汇总整合 → 生成架构/API/数据模型总览
4. 需求反向推断 → brainstorming，从 KB 推断基线需求文档
5. 门禁校验 → G1.1-G1.5
6. Git 提交 → commit 格式 `[devloop] stage:1 action:reverse ...`

**后置条件**: knowledge-base/ 目录 + requirements/baseline/ 目录完成，并提交至 `kb/` 分支
**异常处理**: 门禁失败 → 自动修复或暂停等待人工；codegraph 未索引 → 提示先运行 codegraph index

### 1.2 阶段二：需求摄入 (Requirements Intake)

**触发条件**: 外部需求输入 / 阶段五回环 / 手动创建
**前置条件**: 知识库已就绪（.manifest.yaml 存在）
**输入**: 外部需求（用户故事/功能请求/Bug报告）+ 知识库
**输出**: 结构化需求文档（存入 requirements/in-progress/）

**流程步骤**:
1. 输入解析 → 支持自然语言、用户故事模板、Bug 报告、技术规格四种格式
2. KB 上下文加载 → 关键词匹配 → 索引查询 → 加载 Top-N 条目
3. 影响分析 → brainstorming，生成影响分析报告（涉及模块/数据模型/API/风险）
4. 非功能需求强制评估 → NFR checklist，7 个维度触发条件驱动
5. 需求文档生成 → openspec-propose 创建 OpenSpec Change
6. 门禁校验 → G2.1-G2.5
7. 人工确认 → Level 3 默认需要确认需求内容
8. Git 提交 → status: proposed

**后置条件**: 需求文档存入 requirements/in-progress/，关联 OpenSpec change
**异常处理**: 输入无效（G2.1 reject）→ 返回错误信息；无匹配 KB 模块 → 标记为新模块

### 1.3 阶段三：代码生成 (Code Generation)

**触发条件**: 阶段二完成 / 自动拉取未完成需求
**前置条件**: 需求文档 status in [proposed, designed, planned]
**输入**: 需求文档 + 知识库 + 现有代码
**输出**: 代码变更

**流程步骤**:
1. 自动拉取需求 → 从 Git 扫描 requirements/in-progress/，过滤 status != completed, blocked
2. 优先级排序 → 按 priority/created/age 综合评分
3. Comet Design → comet-handoff → brainstorming → design.md → comet-guard
4. 人工确认设计方案（Level 3/2）
5. Comet Plan → writing-plans → plan.md + tasks.md → comet-guard
6. 人工确认实现计划（Level 3 仅）
7. 选择 Build 模式 → 小需求（< 5 tasks）用 executing-plans，中大型用 subagent-driven-development
8. Build Loop → 每个 Task：实现 → TDD → 双审查（Spec Compliance + Code Quality）→ Git Commit
9. 门禁校验 → G3.1-G3.4

**后置条件**: 代码变更就绪，等待阶段四验证
**异常处理**: 需求不可构建（G3.1）→ 标记 blocked，跳过；Design guard 失败 → 重新 brainstorm；Build 修复最多 3 轮

### 1.4 阶段四：验证与交付 (Verification & Delivery)

**触发条件**: 阶段三完成
**前置条件**: 所有 Task 已完成并提交
**输入**: 代码变更
**输出**: 已验证的提交 + 需求状态更新

**流程步骤**:
1. 确定验证级别 → comet-state scale
2. 自动化验证 → 运行测试套件 + 构建 + Lint + 类型检查（G4.1）
3. 代码审查 → code-review（G4.2）
4. 修复循环 → 失败则 systematic-debugging，最多 3 次（G4.3）
5. 最终门禁 → G4.4
6. Git Commit 代码 → `[devloop] stage:4 ...`
7. 更新需求状态 → status: completed
8. 移动需求文档 → in-progress/ → completed/

**后置条件**: 代码已提交，需求状态更新，进入阶段五
**异常处理**: 连续 3 次验证失败 → 暂停，让用户选择继续/接受偏差/放弃

### 1.5 阶段五：闭环回写 (Loop Back)

**触发条件**: 阶段四完成
**前置条件**: 需求状态已更新
**输入**: 完成的需求记录
**输出**: 触发新一轮需求处理

**流程步骤**:
1. 扫描 requirements/in-progress/，过滤状态有效的需求
2. 状态一致性检查（G5.1）
3. 优先级排序
4. 取最高优先级需求，根据当前状态路由：
   - proposed → 阶段二（从需求完善开始）
   - designed → 阶段三（从 Plan 开始）
   - planned → 阶段三（从 Build 开始）
5. 队列为空 → 进入监听模式（Cron / Webhook / 手动）

---

## 2. 硬/软门禁分离策略

**核心原则**: 事实性检查（脚本）可阻断流程，语义性检查（LLM）仅告警不阻断。

### 两阶段执行模型

```
第一阶段: 硬门禁（脚本自动判定）
  全部通过 ──→ 第二阶段: 软门禁（LLM 语义评估）
  ↓                      ↓
  有失败                 有警告 → 记录 .drift-report.yaml
  按 on_fail 策略处理     流程继续，不阻断
```

### 硬门禁规则
- 只能使用脚本自动判定（`test -f`、`grep -c`、退出码等）
- on_fail 可选: block_and_retry | warn_and_continue | reject | pause_for_human
- 结果决定流程是否阻断

### 软门禁规则
- 委托 LLM agent 评估，输出结构化结果（passed/reason/suggestion）
- on_fail 仅限: warn_and_continue | pause_for_human
- NEVER 使用 block_and_retry
- 警告写入 `.drift-report.yaml` 的 `soft_gate_warnings` 字段
- 阶段总门禁展示警告但不阻断

### 门禁分层

```
L3: 阶段总门禁 (Stage Gate) — 产物完备，可进入下一阶段
L2: 产物门禁 (Artifact Gate) — 单项产物质量合格
L1: 操作门禁 (Operation Gate) — 单步操作执行正确
```

---

## 3. 基线需求用途限制（reference_only）

### 规则

1. 所有基线需求（从代码反向推断的）frontmatter 必须包含 `source: auto-inferred` 和 `usage_restriction: reference_only`
2. `reference_only` 含义：可在阶段二影响分析时作为背景知识，但不作为阶段三设计方案的强制约束
3. 新需求与基线需求冲突时，以新需求为准（基线需求不能否决新设计）
4. 门禁 G2.4 检查 `source` 和 `usage_restriction` 字段存在性
5. 门禁 G3.2 检查设计方案未将 `usage_restriction: reference_only` 的需求作为强制约束

### 目的
防止低质量的逆向推断结果污染下游正向生成过程，确保新设计不被过时或错误的逆向需求所约束。

---

## 4. KB 漂移检测与保鲜触发规则

### 漂移检测时机

| 时机 | 触发者 | 范围 |
|------|--------|------|
| 阶段一逆向 | devloop-reverse | 全量模块 |
| 阶段四验证后 | devloop-verify | 仅受影响的模块 |
| 每周定时 | Cron "0 2 * * 0" | 全量扫描，report_only |
| 每月定时 | Cron "0 3 1 * *" | 全量扫描，auto_fix |
| 手动 | `/devloop kb-fresh` | 指定范围 |

### 阻断规则

- 任意模块 drift_score > 0.3 → 阻断阶段三，强制触发该模块的增量逆向
- 全局平均 drift_score > 0.2 → 阻断所有阶段，建议全量逆向重建
- 低 confidence (<0.7) 的条目被 >3 个需求引用 → 阻断，要求人工审核

### 保鲜流程

1. `git diff` 检测自上次 KB 更新以来的代码变更文件
2. 映射变更文件到受影响模块
3. 对每个受影响模块：codegraph 获取当前符号 → 对比 KB 条目 → 标记差异
4. 生成漂移报告 (`.drift-report.yaml`)
5. auto_fixable=true → 自动更新 KB；auto_fixable=false → 等待人工

### Git 变更检测原理

使用 `git diff <LAST_KB_UPDATE> HEAD -- src/`，捕获两个 commit 间的所有变更，天然处理同日多次变更。保鲜自身产生的 commit 自动推进基线。

---

## 5. 上下文预算分配策略

### 原则

- 阶段一：细粒度模块拆分，单模块 < 50K tokens
- 阶段二/三：索引式按需加载，不加载完整 KB

### 拆分粒度

| 项目大小 | 拆分策略 |
|---------|---------|
| 小型（< 100 文件） | 按顶层目录拆分 |
| 中型（100-500 文件） | 按二级目录拆分 |
| 大型（500+ 文件） | 按功能域拆分 |

### 索引式加载流程

1. 需求输入 → 关键词提取（模块名、函数名、数据实体）
2. 关键词 → KB 索引查询（.manifest.yaml + frontmatter 字段）
3. 匹配结果排序（按相关度）
4. 加载 Top-N 相关条目（N 由上下文预算决定）
5. 超预算 → 加载摘要层（overview.md 优先于 api.md）

### 上下文预算分配（200K 窗口）

| 类别 | 预算 | 说明 |
|------|------|------|
| 系统指令 + Skill 定义 | 20K | 固定开销 |
| 需求文档 | 5K | 当前需求 |
| KB 相关条目 | 80K | 按需索引加载 |
| 相关源代码 | 50K | codegraph 查询 |
| 生成产物 | 30K | design/plan/task |
| 缓冲 | 15K | 余量 |

---

## 6. 确认分级和中断恢复流程

### 确认级别

| 级别 | 确认点 | 自动推进 |
|------|--------|---------|
| Level 3（默认） | 需求内容 + 设计方案 + 实现计划 + 验证失败决策 | 无自动推进 |
| Level 2 | 设计方案 + 验证失败决策 | 需求内容、实现计划自动 |
| Level 1 | 仅验证失败决策 | 全自动至阶段四 |

### 中断恢复流程

```
启动时:
  1. 读取 .comet/loop-state.yaml
  2. 如有活跃流程:
     a. 显示中断位置和上下文摘要
     b. 运行 comet-state check <name> <phase> --recover
     c. 按 Recovery action 继续 (resume_from_step / replay_missing / restart_stage)
  3. 如无活跃流程 → 正常启动

恢复优先级:
  1. pending_confirmations → 优先处理
  2. errors 未解决 → 先解决错误
  3. 否则 → 从 current_step 继续
```

---

## 7. 验证策略

### 输出正确性聚焦

验证只关注输出的正确性，不要求环境一致性：

**关注**: 单元测试通过 / 集成测试通过 / Lint Type Check 零错误 / 构建成功 / 功能符合 spec
**不关注**: 机器环境一致性 / 运行路径一致性 / Lint 规则版本锁定 / 构建工具版本一致

### 测试分层

| 层级 | 范围 | 门禁级别 |
|------|------|---------|
| L1 单元测试 | 函数/方法级别 | 阻塞级 |
| L2 集成测试 | 模块间交互 | 阻塞级 |
| L3 端到端测试 | 需求验收标准 | 驱动级 |

不要求测试覆盖率数值，但要求新代码必须有对应测试。
