---
module: comet-scripts
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
---

# comet-scripts — 业务逻辑

## 核心流程

### comet-state 状态初始化流程
- **触发条件**: `comet-state init <change-name>`
- **前置条件**: 无（首次运行）/ 存在旧状态文件时自动备份
- **流程步骤**:
  1. 检查参数 `change-name`，为空则报错退出
  2. 生成 ISO 8601 UTC 时间戳
  3. 如果 `loop-state.yaml` 已存在，备份为 `loop-state.yaml.bak.<timestamp>`
  4. 写入新状态文件，包含 version、change_name、checkpoint、completed_steps、gate_results、pending_confirmations、errors 等字段
  5. 输出确认信息
- **后置条件**: `loop-state.yaml` 文件已创建且包含完整初始结构
- **异常处理**:
  - 缺少参数 → 输出 Usage 信息，返回 1

### comet-state checkpoint 保存流程
- **触发条件**: `comet-state checkpoint <stage> <step> "<description>"`
- **前置条件**: `loop-state.yaml` 必须存在
- **流程步骤**:
  1. 验证参数（stage 和 step 非空）
  2. 检查状态文件是否存在
  3. 生成时间戳，获取当前 git commit hash
  4. 更新 checkpoint 区域字段：stage、step、last_action、timestamp、git_commit
  5. 同步更新全局字段：current_stage、current_step、updated
  6. 输出 checkpoint 摘要信息
- **后置条件**: checkpoint 和 current_stage/current_step 已更新
- **异常处理**:
  - 状态文件不存在 → 报错返回 1
  - 缺少参数 → 输出 Usage 信息，返回 1

### comet-state 恢复检查流程
- **触发条件**: `comet-state check <name> <phase> [--recover]`
- **前置条件**: 无（状态文件不存在时输出 fresh_start 建议）
- **流程步骤**:
  1. 如果状态文件不存在，输出 "状态文件不存在" 和 "Recovery action: fresh_start"
  2. 读取当前 stage、step、pending confirmations 数、failed gates 数
  3. 输出恢复检查报告
  4. 如果 `--recover` 标记启用，按条件输出恢复建议：
     - 有待确认项 → 建议加载 devloop-resume Skill
     - 有失败门禁 → 建议检查 gate_results
     - 有当前步骤 → 输出 resume_from_step
     - 有当前阶段 → 输出 resume_from_stage
- **后置条件**: 恢复信息已输出
- **异常处理**: 无（所有路径正常返回）

### devloop-guard 门禁检查流程
- **触发条件**: `devloop-guard check <stage>`
- **前置条件**: `guard-config.yaml` 必须存在
- **流程步骤**:
  1. 标准化 stage 参数（数字转 `stage-N` 格式）
  2. 检查 `guard-config.yaml` 是否存在
  3. 使用 `parse_gates_simple` 解析所有门禁配置
  4. 遍历门禁，仅处理属于当前 stage 的门禁：
     - **hard**: 调用 `run_hard_check` 执行脚本检查。输出 PASS 或 FAIL
     - **soft**: 跳过（标注 `[SKIP] — LLM 评估阶段`），收集到软门禁列表
     - **mixed**: 先执行 hard 部分（调用 `run_hard_check`），再跳过 soft 部分，两者都收集
  5. 汇总硬门禁结果（passed/failed 计数）
  6. 如果有 soft 门禁，生成软门禁待办文件 `soft-gates-pending.json`
  7. 输出最终判定：ALL CHECKS PASSED 或 GATE CHECK FAILED
- **后置条件**: 硬门禁通过/失败状态已记录；软门禁待办清单已写入
- **异常处理**:
  - `guard-config.yaml` 不存在 → 报错返回 1
  - 缺少 stage 参数 → 输出 Usage 信息
  - 硬门禁失败 → 列出失败门禁，返回 1

### devloop-guard 门禁配置验证流程
- **触发条件**: `devloop-guard validate-config`
- **前置条件**: `guard-config.yaml` 必须存在
- **流程步骤**:
  1. 解析所有门禁配置
  2. 对每个门禁检查类型/on_fail 约束：
     - soft 类型不能使用 on_fail=block_and_retry
     - soft 类型不能使用 on_fail=reject
  3. 统计 hard/soft/mixed 门禁数量
  4. 输出统计结果和验证结论
- **后置条件**: 无文件状态变更
- **异常处理**:
  - 有违规 → 列出违规项，返回 1
  - 所有通过 → 输出验证通过

### 硬门禁检查实现（run_hard_check）
- **触发条件**: `devloop-guard check` 遇到 hard 或 mixed 类型门禁
- **前置条件**: 无
- **流程步骤**: 按 `gate_id` 分发：
  - **G1.1** 模块覆盖率: 统计 `knowledge-base/modules/` 目录数与 `src/` 目录数之比 >= 80%
  - **G1.2** KB条目完整性: 每个模块下必须存在 `overview.md` 和 `api.md`
  - **G1.3** KB内部一致性(hard): 检查 wiki-link 引用目标是否存在
  - **G1.4** 需求反向覆盖率: 统计 baseline 需求数与模块数
  - **G1.5/G2.5/G4.4** 阶段总门禁: 汇总判定（由前序门禁结果决定）
  - **G2.1** 输入有效性: 检查需求文档 frontmatter 中 title 和 description 存在
  - **G2.3** 影响分析结构: 结构检查（当前为占位）
  - **G2.4** 需求文档结构: 检查 frontmatter 包含 id/title/type/priority/status
  - **G3.1** 需求可构建性: （当前为占位）
  - **G3.2** Design阶段质量(hard): 检查 design.md 存在
  - **G3.3** Plan阶段质量(hard): 检查 task 编号唯一性（当前为占位）
  - **G3.4** Build阶段质量: 检查 Build 完成状态（当前为占位）
  - **G4.1** 自动化验证: （当前为占位）
  - **G4.2** 代码审查(hard): 检查审查报告存在（当前为占位）
  - **G4.3** 修复循环限制: （当前为占位）
  - **G5.1** 闭环一致性: （当前为占位）
- **后置条件**: 输出检查结果
- **异常处理**: 未实现的检查返回 0（不阻塞）

## 业务规则

- **软门禁失败策略限制**: soft 类型门禁只能使用 `warn_and_continue` 或 `pause_for_human`，禁止使用 `block_and_retry` 或 `reject`
- **自动备份**: 状态文件初始化前自动备份旧文件
- **自动时间戳**: 所有状态更新操作自动更新 `updated` 时间戳
- **门禁汇总判定**: 阶段总门禁（G1.5/G2.5/G4.4）的判定由前序门禁结果决定
- **软门禁不阻断**: soft 门禁即使失败也只输出 WARN，不会阻断流程
- **硬门禁可阻断**: hard 门禁使用 block_and_retry 策略时可以阻断流程并触发自动重试（最多 3 次）

## 副作用

- 写入 `.comet/loop-state.yaml` — 主状态文件，所有子命令都会读写此文件
- 写入 `.comet/loop-state.yaml.bak.<timestamp>` — 备份旧状态文件（init 时）
- 写入 `.comet/gate-results/soft-gates-pending.json` — 软门禁待办清单（check 时）
- 写入 `.comet/gate-results/` 目录 — 软门禁评估结果目录
- git commit hash 检索 — checkpoint 时读取但不修改 git 状态
