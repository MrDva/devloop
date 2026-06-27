---
module: comet-scripts
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
---

# comet-scripts — 数据模型

## 实体定义

### loop-state.yaml
- **类型**: YAML file
- **描述**: DevLoop 运行时状态文件，位于 `.comet/loop-state.yaml`。由 `comet-state init` 创建，通过 `comet-state` 各子命令操作
- **字段**:
  - `devloop.version` (string) **必需**: 状态文件版本号，固定为 `"1.0"`
  - `devloop.updated` (string) **必需**: 最后更新时间，ISO 8601 UTC 格式，每次 set/write 自动更新
  - `devloop.change_name` (string) **必需**: 当前变更名称
  - `devloop.current_stage` (integer) **必需**: 当前阶段号（0-5）
  - `devloop.current_step` (string): 当前步骤标识（如 `"1.3"`）
  - `devloop.active_requirement.id` (string): 活跃需求 ID
  - `devloop.active_requirement.title` (string): 活跃需求标题
  - `devloop.active_requirement.change_name` (string): 活跃需求所属变更名称
  - `devloop.checkpoint.stage` (integer): 断点阶段号
  - `devloop.checkpoint.step` (string): 断点步骤标识
  - `devloop.checkpoint.last_action` (string): 最后操作描述
  - `devloop.checkpoint.timestamp` (string): 断点时间戳，ISO 8601 UTC
  - `devloop.checkpoint.git_commit` (string): 断点时 HEAD commit hash（short）
  - `devloop.completed_steps` (map): 已完成步骤记录，键为 `stage-N`，值为步骤列表
  - `devloop.gate_results` (map): 门禁结果记录，键为 gate ID，值为 `{passed, timestamp, message}`
  - `devloop.pending_confirmations` (array): 待确认项列表，每项包含 `type`, `req`, `message`, `timestamp`
  - `devloop.errors` (array): 错误记录列表
- **约束**:
  - 自动生成，禁止手动编辑 — 必须通过 `comet-state` 命令操作
  - `current_stage` 范围 0-5
  - `version` 固定为 `"1.0"`
- **关联**:
  - → guard-config.yaml: `current_stage` 对应 guard-config.yaml 中的 stage-N 门禁组

### guard-config.yaml
- **类型**: YAML file
- **描述**: DevLoop 门禁配置文件，位于 `.comet/guard-config.yaml`。定义 5 个阶段共 18 个门禁的检查规则和失败处理策略
- **字段**:
  - `gates` (map) **必需**: 门禁配置根节点
  - `gates.stage-N` (list) **必需**: 阶段 N（1-5）的门禁列表
  - `gates.stage-N[].id` (string) **必需**: 门禁 ID（如 `G1.1`、`G2.3`）
  - `gates.stage-N[].name` (string) **必需**: 门禁名称
  - `gates.stage-N[].type` (string) **必需**: 门禁类型，可选值 `hard`、`soft`、`mixed`
  - `gates.stage-N[].stage` (integer) **必需**: 所属阶段
  - `gates.stage-N[].description` (string): 门禁描述
  - `gates.stage-N[].checks` (list): hard 类型门禁的检查规则列表
  - `gates.stage-N[].checks[].type` (string): 检查类型（`file_count`、`file_exists`、`field_nonempty` 等）
  - `gates.stage-N[].checks[].rule` (string): 检查规则表达式
  - `gates.stage-N[].checks[].detail` (string): 检查规则详细说明
  - `gates.stage-N[].hard` (list): mixed 类型门禁的硬门禁检查列表
  - `gates.stage-N[].soft` (list): mixed 类型门禁的软门禁检查列表
  - `gates.stage-N[].on_fail` (string/map) **必需**: 门禁失败处理策略
    - string 值：`block_and_retry`、`warn_and_continue`、`reject`、`pause_for_human`
    - map 值（mixed 类型）：`{hard: <策略>, soft: <策略>}`
- **约束**:
  - soft 类型门禁只能使用 `warn_and_continue` 或 `pause_for_human`
  - soft 类型门禁禁止使用 `block_and_retry` 和 `reject`
  - 门禁 ID 格式为 `G{阶段}.{序号}`（如 G1.1、G2.3）
- **关联**:
  - → loop-state.yaml: stage-N 被 `current_stage` 引用以选择执行哪组门禁
  - → `.comet/gate-results/soft-gates-pending.json`: devloop-guard check 生成的软门禁待办清单

### gate-results JSON
- **类型**: JSON file
- **描述**: 软门禁待办清单文件，由 `devloop-guard check` 在遇到 soft/mixed 门禁时生成。位于 `.comet/gate-results/soft-gates-pending.json`
- **字段**:
  - `stage` (string): 当前阶段标识（如 `"stage-1"`）
  - `generated_at` (string): 生成时间戳，ISO 8601 UTC
  - `soft_gates_pending` (array of string): 待评估的软门禁 ID 列表
  - `instruction` (string): 评估指引文本

### loop-state.yaml 内嵌结构

#### completed_steps 条目
```
stage-1:
  - step: 1.1
    status: completed
    timestamp: "2026-06-27T..."
```

#### gate_results 条目
```
G1.1: { passed: true, timestamp: "2026-06-27T...", message: "模块覆盖: 5/6 (83%)" }
```

#### pending_confirmations 条目
```
- type: human_review
  req: design_approval
  message: "请确认设计文档"
  timestamp: "2026-06-27T..."
```

## 状态转换

### DevLoop 生命周期
```
init → stage-0
  │
  ▼
stage-1 (逆向分析) ─→ 门禁检查 G1.1-G1.5 ─→ pass → stage-2
  │                                              │
  └── fail → block_and_retry / warn_and_continue  │
                                                 ▼
stage-2 (需求摄入) ─→ 门禁检查 G2.1-G2.5 ─→ pass → stage-3
                                                 │
                                                 ▼
stage-3 (代码生成) ─→ 门禁检查 G3.1-G3.4 ─→ pass → stage-4
                                                 │
                                                 ▼
stage-4 (验证交付) ─→ 门禁检查 G4.1-G4.4 ─→ pass → stage-5
                                                 │
                                                 ▼
stage-5 (闭环) ─→ 门禁检查 G5.1 ─→ pass → archive
```

### 步骤完成状态
```
pending → completed
```
- 步骤通过 `complete-step` 命令标记为 completed
- 存储在 `completed_steps` 的 `stage-N` 列表中

### 门禁结果状态
```
未执行 → pass / fail
```
- 硬门禁通过 `run_hard_check` 脚本执行，退出码 0 为 pass，非 0 为 fail
- 软门禁由 LLM 评估，结果写入 `.comet/gate-results/soft-<gate-id>.json`
- 结果通过 `gate-result` 记录到 `gate_results` 映射中
