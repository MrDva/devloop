---
id: REQ-001
title: 运行时状态管理
module: comet-scripts
status: baseline
inferred_from:
  - .comet/scripts/comet-state
  - .comet/scripts/devloop-guard
confidence: high
source: auto-inferred
usage_restriction: reference_only
created: "2026-06-27T14:00:00Z"
---

> ⚠️ **AUTO-INFERRED BASELINE** | confidence: high | source: code reverse-engineering
>
> This requirement was inferred from existing code implementation, NOT from original design documents or specifications. It may not reflect original design intent.
>
> **Usage restriction (`reference_only`)**: This baseline requirement serves as **documentation reference only**. It MUST NOT be used as a mandatory constraint when designing new features in Stage 3. If a new requirement conflicts with a baseline requirement, the baseline requirement gives way.

# REQ-001: 运行时状态管理

## 推断来源

从以下代码反向推断：
- `.comet/scripts/comet-state` (14444 bytes, 14 子命令)
- `.comet/scripts/devloop-guard` (20079 bytes, 3 子命令)

## 功能描述

系统应提供运行时状态管理能力，支持：

1. **状态文件初始化**: 创建 `loop-state.yaml` 文件，包含 devloop 版本、变更名称、当前阶段等字段
2. **状态读写**: 用 `get`/`set` 子命令读写任意字段值，支持嵌套键路径（如 `active_requirement.id`）
3. **断点保存**: 每个操作步骤结束后调用 `checkpoint` 保存当前进度，记录 stage/step/描述/时间戳/git commit
4. **步骤标记**: `complete-step` 标记步骤完成，防止重复执行
5. **门禁记录**: `gate-result` 记录每个门禁的 pass/fail 状态和详情
6. **待确认项管理**: `add-pending`/`list-pending` 管理人工确认队列
7. **中断恢复**: `check --recover` 读取状态文件，输出恢复动作建议（resume_from_step / resume_from_pending / gate_failures_present）
8. **状态摘要**: `scale` 显示当前进度、完成步骤数、待确认项数、失败门禁列表
9. **yq 降级**: yq 不可用时自动降级为 grep/sed/awk 模式操作 YAML

## 当前实现

- 脚本: `.comet/scripts/comet-state` (bash)
- 状态文件: `.comet/loop-state.yaml`
- 配置文件: `.comet/config.yaml`
- 恢复逻辑: check 子命令分析状态文件和已完成步骤，判断中断位置
- 备份: init 命令在覆盖旧状态前自动备份为 `.bak`
