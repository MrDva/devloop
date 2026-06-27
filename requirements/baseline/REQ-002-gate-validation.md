---
id: REQ-002
title: 门禁校验系统
module: comet-scripts
status: baseline
inferred_from:
  - .comet/scripts/devloop-guard
  - .comet/guard-config.yaml
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

# REQ-002: 门禁校验系统

## 推断来源

从以下代码反向推断：
- `.comet/scripts/devloop-guard` (20079 bytes)
- `.comet/guard-config.yaml` (14059 bytes, 18 门禁定义)

## 功能描述

系统应提供分阶段的门禁校验能力，支持：

1. **18 个门禁**: 覆盖 5 个阶段（G1.1-G1.5 阶段一, G2.1-G2.5 阶段二, G3.1-G3.4 阶段三, G4.1-G4.4 阶段四, G5.1 阶段五）
2. **硬/软门禁分离**: 
   - hard: 脚本自动判定（文件存在性、计数比较、引用有效性），可 BLOCK
   - soft: LLM 语义评估（一致性、完整性判断），仅 WARN 不阻断
   - mixed: 两层分别判定，hard 层可 BLOCK，soft 层仅 WARN
3. **四种失败处理策略**: block_and_retry（阻断+重试）、warn_and_continue（警告+继续）、reject（拒绝）、pause_for_human（暂停等人工）
4. **软门禁机制**: 硬门禁全部通过后输出 `SOFT_GATES_PENDING` 信号，委托 LLM 逐项评估，结果写入 `.comet/gate-results/soft-<id>.json`
5. **配置验证**: `validate-config` 确保 soft 类型门禁不使用 BLOCK 级别阻断策略
6. **汇总判定**: 阶段总门禁（如 G1.5）汇总前序门禁结果，执行 gate_aggregate 规则

## 当前实现

- 脚本: `.comet/scripts/devloop-guard` (bash)
- 配置: `.comet/guard-config.yaml`
- 阶段一: G1.1（模块覆盖率）、G1.2（KB完整性）、G1.3（一致性）、G1.4（需求覆盖率）、G1.5（总门禁）
- 硬门禁实现: run_hard_check 函数按 check type 分发到具体检查逻辑
- 软门禁: 输出待办清单到 `.comet/gate-results/soft-gates-pending.json`
