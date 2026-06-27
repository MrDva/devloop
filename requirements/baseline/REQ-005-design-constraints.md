---
id: REQ-005
title: 设计规范与架构约束
module: design-docs
status: baseline
inferred_from:
  - design/01-overview.md
  - design/02-detailed-design.md
  - design/05-implementation-tasks.md
confidence: medium
source: auto-inferred
usage_restriction: reference_only
created: "2026-06-27T14:00:00Z"
---

> ⚠️ **AUTO-INFERRED BASELINE** | confidence: medium | source: code reverse-engineering
>
> This requirement was inferred from existing code implementation, NOT from original design documents or specifications. It may not reflect original design intent.
>
> **Usage restriction (`reference_only`)**: This baseline requirement serves as **documentation reference only**. It MUST NOT be used as a mandatory constraint when designing new features in Stage 3. If a new requirement conflicts with a baseline requirement, the baseline requirement gives way.

# REQ-005: 设计规范与架构约束

## 推断来源

从以下设计文档反向推断：
- `design/02-detailed-design.md` (1800+ 行完整设计)
- `design/05-implementation-tasks.md` (838 行任务清单)
- `design/01-overview.md`, `design/03-flowcharts.md`

## 功能描述

系统应遵循以下设计规范和架构约束：

1. **五阶段架构**: 逆向分析→需求摄入→代码生成→验证交付→闭环回写，阶段间通过门禁衔
2. **薄编排层原则**: 新建 Skill 只做编排，不重复实现已有 Skill 的功能，最大化复用 14 个已有 Skill
3. **硬/软门禁分离**: 硬门禁脚本判定可阻断，软门禁 LLM 评估仅警告，mixed 两层分别判定。软门禁禁止使用 block_and_retry 策略
4. **KB 防御机制**: 所有 KB 条目含 `source`/`confidence`/`drift_score` frontmatter；三级信任体系（Trusted/Tentative/Untrusted）；漂移阻断规则（drift_score > 0.3 阻断阶段三）
5. **基线需求用途限制**: 所有从代码反推的需求标记 `usage_restriction: reference_only`，仅供背景参考，不得作为阶段三设计的强制约束
6. **上下文窗口管理**: 单模块 KB < 50K tokens；阶段二/三索引式按需加载，超出预算时摘要层降级
7. **验证聚焦**: 输出正确性聚焦（测试通过=正确），不要求环境一致性
8. **Git 驱动的 KB 保鲜**: 利用 `git diff` 检测代码变更 → 映射受影响模块 → 增量逆向或全量扫描

## 当前实现

- `design/02-detailed-design.md` 记录了完整的架构设计
- `design/05-implementation-tasks.md` 将设计细化为 20 个具体任务（Phase 0-6）
- `templates/` 定义了所有产出物格式
- `.comet/guard-config.yaml` 实现了 18 个门禁
- 部分约束已通过脚本强制执行（如 devloop-guard validate-config 检查软门禁策略）
