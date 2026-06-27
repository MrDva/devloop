---
module: comet-scripts
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
---

# comet-scripts — 依赖关系

## 上游依赖（本模块依赖的）

### yq
- **依赖类型**: 可选工具
- **依赖内容**: YAML 处理，用于读写 `loop-state.yaml` 和解析 `guard-config.yaml`
- **耦合度**: 弱（可选降级）
- **接口**: `yq ".${key}" "$STATE_FILE"` / `yq -i ".${key} = \"${value}\"" "$STATE_FILE"`

### jq
- **依赖类型**: 可选工具
- **依赖内容**: JSON 处理（用于生成 `soft-gates-pending.json`）
- **耦合度**: 弱（未硬性要求）
- **接口**: 未直接在代码中使用（由 `has_jq` 检测但未在核心逻辑中使用）

### bash 4.x
- **依赖类型**: 运行时
- **依赖内容**: 脚本执行环境，需要 associative arrays、conditional expressions 等 bash 4+ 特性
- **耦合度**: 强（运行必须）
- **接口**: 标准 bash 调用

### git
- **依赖类型**: 工具
- **依赖内容**: 在 checkpoint 时获取当前 HEAD commit hash
- **耦合度**: 弱（git 不可用时回退到 `N/A`）
- **接口**: `git rev-parse --short HEAD`

### grep / sed / awk
- **依赖类型**: 运行时依赖
- **依赖内容**: YAML 降级解析（yq 不可用时）、门禁配置解析（awk）
- **耦合度**: 强（解析核心实现）
- **接口**: 标准 POSIX 命令行工具

### python3
- **依赖类型**: 可选工具（devloop-guard 使用）
- **依赖内容**: 备选的 YAML 解析方式（`parse_gates_full` 函数）
- **耦合度**: 弱（仅在 `parse_gates_full` 中使用，主流程使用 awk 实现 `parse_gates_simple`）
- **接口**: python3 脚本调用

## 下游依赖（依赖本模块的）

### devloop-skills
- **依赖类型**: 编排调用
- **依赖内容**: 所有 DevLoop 编排 Skill（包括 executing-plans、subagent-driven-development、verification-before-completion 等 6 个编排 Skill）都调用 comet-state 进行状态管理，调用 devloop-guard 进行门禁校验
- **耦合度**: 强（编排 Skill 无法脱离状态管理和门禁运行）

### 其他 Skill
- **依赖类型**: 间接依赖
- **依赖内容**: brainstorming、writing-plans、test-driven-development 等 Skill 在 DevLoop 上下文中运行，通过编排 Skill 间接依赖 comet-scripts 管理的流程状态
- **耦合度**: 中

## 依赖图
```
┌─────────────────────────────────────────────────────────────┐
│                     comet-scripts                           │
│  ┌──────────────────────┐  ┌────────────────────────────┐   │
│  │    comet-state       │  │    devloop-guard           │   │
│  │  (状态管理, 14命令)   │  │  (门禁引擎, 3命令)         │   │
│  └──────┬───────────────┘  └──────┬─────────────────────┘   │
│         │                         │                         │
│         ▼                         ▼                         │
│  ┌──────────────────────────────────────────────┐           │
│  │  .comet/loop-state.yaml                      │           │
│  │  .comet/guard-config.yaml                    │           │
│  │  .comet/gate-results/*.json                  │           │
│  └──────────────────────────────────────────────┘           │
└────────────┬──────────────────────────────┬─────────────────┘
             │                              │
             ▼                              ▼
  ┌────────────────────┐      ┌──────────────────────────┐
  │  外部工具            │      │  下游消费方               │
  │  yq (可选)          │      │  devloop-skills (6个)    │
  │  git               │      │  brainstorming           │
  │  grep/sed/awk      │      │  writing-plans           │
  │  bash 4+           │      │  tdd / sdd               │
  │  python3 (可选)     │      │  验证 Skill              │
  └────────────────────┘      └──────────────────────────┘

依赖方向: 上游 → comet-scripts → 下游
```

## 循环依赖
✅ 未检测到循环依赖

comet-scripts 是 DevLoop 体系中的底层基础设施模块，仅依赖外部工具（bash、yq、git 等），不依赖任何其他 KB 模块。下游消费方（devloop-skills、其他 Skill）单向依赖 comet-scripts 提供的状态管理和门禁功能，不存在反向依赖关系。

## 调用关系

### comet-state 被调用模式
```
编排 Skill → comet-state init <change-name>
编排 Skill → comet-state checkpoint <stage> <step> <desc>
编排 Skill → comet-state complete-step <stage> <step>
编排 Skill → comet-state gate-result <gate-id> <pass|fail>
编排 Skill → comet-state check <name> <phase> [--recover]
编排 Skill → comet-state scale [name]
```

### devloop-guard 被调用模式
```
编排 Skill / verify Skill → devloop-guard check <stage>
编排 Skill → devloop-guard list [--type ...]
编排 Skill → devloop-guard validate-config
```

### 门禁结果回写模式
```
devloop-guard check → 硬门禁: 直接输出
                   → 软门禁: 写入 soft-gates-pending.json
LLM agent → 读取 soft-gates-pending.json → 逐个评估 → 写入 soft-<gate-id>.json
编排 Skill → comet-state gate-result <gate-id> pass/fail → 记录到 loop-state.yaml
```
