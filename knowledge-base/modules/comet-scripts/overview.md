---
module: comet-scripts
path: .comet/scripts/
type: utility
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - .comet/scripts/comet-state
  - .comet/scripts/devloop-guard
  - .comet/guard-config.yaml
---

# comet-scripts — 模块概述

## 职责
DevLoop 运行时状态管理和门禁校验。提供两个核心脚本：`comet-state` 负责 loop-state.yaml 的初始化、读写、checkpoint 和恢复；`devloop-guard` 负责门禁配置管理、硬/软门禁分离执行和软门禁待办清单输出。

## 模块类型
utility

## 关键指标
- 文件数: 3（2 个脚本 + 1 个配置文件）
- 主要语言: Bash
- 估计 Token 数: 4500

## 核心功能
- **comet-state**: 14 个子命令的状态管理脚本，管理 loop-state.yaml 生命周期
- **devloop-guard**: 3 个子命令的门禁执行引擎，运行 18 个门禁检查
- **guard-config.yaml**: 门禁配置文件，定义 5 个阶段共 18 个门禁

## 对外接口
- `comet-state init <change-name>` — 初始化 DevLoop 状态文件
- `comet-state get <key>` — 读取状态字段值
- `comet-state set <key> <value>` — 设置状态字段值
- `comet-state checkpoint <stage> <step> <desc>` — 保存断点
- `comet-state complete-step <stage> <step>` — 标记步骤完成
- `comet-state gate-result <gate-id> <pass|fail> [msg]` — 记录门禁结果
- `comet-state add-pending <type> <req> <msg>` — 添加待确认项
- `comet-state list-pending` — 列出待确认项
- `comet-state check <name> <phase> [--recover]` — 恢复检查
- `comet-state scale [name]` — 显示状态摘要
- `devloop-guard list [--type <hard|soft|mixed>]` — 列出所有门禁
- `devloop-guard check <stage>` — 运行指定阶段硬门禁检查
- `devloop-guard validate-config` — 验证门禁配置

## 依赖关系
- **bash**: 运行时依赖，需要 bash 4.x
- **yq**: 可选，YAML 处理工具；不可用时降级为 grep/sed
- **jq**: 可选，JSON 处理工具
- **git**: 用于 checkpoint 时记录当前 commit hash

## 被依赖关系
- **devloop-skills**: 所有编排 Skill 调用 comet-state 和 devloop-guard 进行状态管理和门禁校验

## 备注
- comet-state 在 yq 不可用时自动降级为 grep/sed/awk 模式
- devloop-guard 支持三种门禁类型：hard（脚本自动判定）、soft（LLM 语义评估）、mixed（两层分别判定）
- soft 门禁只能使用 warn_and_continue 或 pause_for_human，不可使用 block_and_retry 或 reject
- 输出目录为 `.comet/`，gate-results 存储在 `.comet/gate-results/`
