---
module: comet-scripts
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
---

# comet-scripts — API 文档

## 导出符号

### comet-state
- **类型**: bash script
- **描述**: DevLoop 状态管理脚本，管理 loop-state.yaml 的初始化、读写、checkpoint、恢复

#### init
- **签名**: `comet-state init <change-name>`
- **描述**: 初始化状态文件 loop-state.yaml，备份已存在的状态文件
- **参数**:
  - `change-name` (string): 变更名称
- **返回值**: 无，输出确认信息
- **异常**: 缺少参数时输出 Usage 信息并返回 1

#### get
- **签名**: `comet-state get <key>`
- **描述**: 读取状态文件中的字段值。优先使用 yq，不可用时降级为 grep/sed
- **参数**:
  - `key` (string): 字段名（如 `current_stage`、`change_name`）
- **返回值**: 字段值（stdout）
- **异常**: 状态文件不存在时报错返回 1；缺少参数时输出 Usage 信息并返回 1

#### set
- **签名**: `comet-state set <key> <value>`
- **描述**: 设置状态字段值，同时自动更新 `updated` 时间戳
- **参数**:
  - `key` (string): 字段名
  - `value` (string): 字段值
- **返回值**: 无，输出 `key = value`
- **异常**: 状态文件不存在时报错返回 1；缺少参数时输出 Usage 信息并返回 1

#### checkpoint
- **签名**: `comet-state checkpoint <stage> <step> "<description>"`
- **描述**: 保存断点，更新 checkpoint 区域字段（stage、step、last_action、timestamp、git_commit）及相关全局字段
- **参数**:
  - `stage` (string): 阶段号（如 `1`）
  - `step` (string): 步骤标识（如 `1.3`）
  - `description` (string): 断点描述
- **返回值**: 无，输出 checkpoint 信息和 timestamp/commit
- **异常**: 状态文件不存在时报错返回 1；缺少必填参数时输出 Usage 信息并返回 1

#### complete-step
- **签名**: `comet-state complete-step <stage> <step>`
- **描述**: 标记步骤为已完成，追加记录到 completed_steps 区域
- **参数**:
  - `stage` (string): 阶段号
  - `step` (string): 步骤标识
- **返回值**: 无，输出 "Step X.Y 完成"
- **异常**: 状态文件不存在时报错返回 1；缺少参数时输出 Usage 信息并返回 1

#### gate-result
- **签名**: `comet-state gate-result <gate-id> <pass|fail> [message]`
- **描述**: 记录门禁执行结果到 gate_results 区域
- **参数**:
  - `gate-id` (string): 门禁 ID（如 `G1.1`）
  - `result` (string): 结果，`pass` 或 `fail`
  - `message` (string, 可选): 附加消息
- **返回值**: 无，输出 "Gate ID: PASS/FAIL — message"
- **异常**: 状态文件不存在时报错返回 1

#### add-pending
- **签名**: `comet-state add-pending <type> <req> "<message>"`
- **描述**: 添加待确认项到 pending_confirmations 列表
- **参数**:
  - `type` (string): 待确认类型
  - `req` (string): 请求标识
  - `message` (string): 待确认消息
- **返回值**: 无，输出 "待确认: [type] req — message"
- **异常**: 状态文件不存在时报错返回 1

#### list-pending
- **签名**: `comet-state list-pending`
- **描述**: 列出所有待确认项
- **参数**: 无
- **返回值**: 待确认项列表（stdout），无待确认项时输出 "(无)"

#### check
- **签名**: `comet-state check <name> <phase> [--recover]`
- **描述**: 恢复检查，输出当前状态摘要和恢复建议
- **参数**:
  - `name` (string): 变更名称
  - `phase` (string): 阶段名称
  - `--recover` (flag, 可选): 启用恢复模式，输出具体恢复建议
- **返回值**: 输出恢复检查报告

#### scale
- **签名**: `comet-state scale [name]`
- **描述**: 显示 DevLoop 状态摘要，包括当前阶段、步骤、活跃需求、已完成步骤、待确认项
- **参数**:
  - `name` (string, 可选): 变更名称
- **返回值**: 状态摘要报告（stdout）

---

### devloop-guard
- **类型**: bash script
- **描述**: DevLoop 门禁执行引擎，管理门禁配置、硬/软门禁分离执行

#### list
- **签名**: `devloop-guard list [--type <hard|soft|mixed>]`
- **描述**: 列出所有门禁或按类型过滤
- **参数**:
  - `--type` (string, 可选): 过滤类型，支持逗号分隔的多类型（如 `soft,mixed`）
- **返回值**: 格式化的门禁清单表格，包含 ID、Name、Type、Stage、On Fail

#### check
- **签名**: `devloop-guard check <stage>`
- **描述**: 运行指定阶段的硬门禁检查。hard 门禁执行脚本检查；soft/mixed 的 soft 部分输出软门禁待办清单
- **参数**:
  - `stage` (string): 阶段标识，支持数字（`1`-`5`）或完整名称（`stage-1`-`stage-5`）
- **返回值**: 门禁检查报告；全部通过返回 0，有失败返回 1

#### validate-config
- **签名**: `devloop-guard validate-config`
- **描述**: 验证门禁配置中软门禁约束（soft 类型不能使用 block_and_retry 或 reject）
- **参数**: 无
- **返回值**: 验证通过返回 0，有违规返回 1

---

## 内部 API

### yaml_get
- **签名**: `yaml_get <file> <key>`
- **用途**: 内部 YAML 读取函数，从 YAML 文件中按字段名提取值。使用 grep/sed 实现，不依赖 yq

### yaml_set
- **签名**: `yaml_set <file> <key> <value>`
- **用途**: 内部 YAML 写入函数，替换或追加指定字段。字段存在时替换整行（保留缩进），不存在时追加到文件末尾

### now_iso
- **签名**: `now_iso`
- **用途**: 生成 ISO 8601 格式的 UTC 时间戳（`date -u +"%Y-%m-%dT%H:%M:%SZ"`）

### has_yq / has_jq
- **签名**: `has_yq` / `has_jq`
- **用途**: 检测系统是否安装 yq/jq 工具，用于决定使用原生工具还是降级模式

### parse_gates_simple
- **签名**: `parse_gates_simple [config-file]`
- **用途**: 使用 awk 解析门禁配置 YAML，输出每行一条 `id|name|type|stage|on_fail` 格式的记录

### run_hard_check
- **签名**: `run_hard_check <gate-id> <gate-name> <gate-stage>`
- **用途**: 执行指定硬门禁的脚本检查。每个门禁 ID 对应一个 case 分支，实现具体的检查逻辑（文件存在性、计数检查、引用一致性等）

### print_usage
- **签名**: `print_usage`（comet-state / devloop-guard 各有一个）
- **用途**: 输出命令帮助信息，包括所有子命令和示例
