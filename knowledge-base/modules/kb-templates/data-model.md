---
module: kb-templates
source: auto-generated
confidence: 1.0
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - templates/
---

# kb-templates — 数据模型

## 1. KB 条目 Frontmatter 字段

所有 5 个 `kb-module-*.md` 模板共享统一的前置元数据格式：

```yaml
---
module: <string>              # 模块名称（如 "kb-templates"）
source: <string>              # 来源（固定为 "auto-generated"）
confidence: <float 0.0-1.0>  # 自动生成置信度（如 0.95）
drift_score: <float 0.0-1.0>  # 与当前代码的漂移程度（如 0.0）
last_updated: <ISO 8601>      # 最后更新时间戳
generated_from:               # 源文件路径列表（仅 overview 模板有）
  - <path>
  - <path>
---
```

### 字段定义

| 字段 | 类型 | 范围 | 必需性 | 说明 |
|------|------|------|--------|------|
| `module` | string | — | **必需** | 模块名称，作为 KB 条目标识 |
| `source` | string | "auto-generated" | **必需** | 固定为 auto-generated，表示由逆向分析自动产生 |
| `confidence` | float | 0.0 - 1.0 | **必需** | 置信度，1.0 表示完全确定，越低表示猜测成分越多 |
| `drift_score` | float | 0.0 - 1.0 | **必需** | 漂移分数，0.0 表示与当前代码完全一致，升高表示代码已变化而 KB 未更新 |
| `last_updated` | string | ISO 8601 | **必需** | 最后更新时间，格式如 "2026-06-27T14:00:00Z" |
| `generated_from` | string[] | — | overview 特有 | 生成该条目的源文件列表，仅 overview.md 使用 |

### 副本间差异

- `kb-module-overview.md` 额外包含 `generated_from` 字段（多个源文件路径）
- 其余 4 个 KB 条目模板（api、data-model、business-logic、dependencies）的 frontmatter 仅含 `module`、`source`、`confidence`、`drift_score`、`last_updated` 五个字段

---

## 2. 基线需求 Frontmatter 字段（requirement-doc.md）

完整需求文档模板（`requirement-doc.md`）的 frontmatter 比 KB 条目更丰富，包含需求管理所需的全部元数据：

```yaml
---
id: <string>                    # 唯一需求 ID（如 REQ-001）
title: <string>                 # 需求标题
type: <enum>                    # feature | bugfix | enhancement | refactor | chore
priority: <enum>                # high | medium | low
status: <enum>                  # 需求状态（见下文状态机）
module: <string>                # 主要影响模块
source: <enum>                  # auto-inferred | user-submitted | external-import
usage_restriction: <enum>       # reference_only | none
kb_context:
  - <string>                    # 相关 KB 文件路径数组
confidence: <enum>              # high | medium | low
created: <ISO 8601>             # 创建时间
updated: <ISO 8601>             # 更新时间
---
```

### 字段详细说明

| 字段 | 类型 | 可选值 | 必需性 | 说明 |
|------|------|--------|--------|------|
| `id` | string | — | **必需** | 全局唯一需求 ID，命名空间如 REQ- |
| `title` | string | — | **必需** | 需求标题 |
| `type` | enum | feature / bugfix / enhancement / refactor / chore | **必需** | 区分功能、缺陷、增强、重构、杂项 |
| `priority` | enum | high / medium / low | **必需** | 优先级 |
| `status` | enum | baseline / proposed / designed / planned / building / verifying / completed / blocked / superseded | **必需** | 需求状态（见状态机） |
| `module` | string | — | **必需** | 主要影响模块 |
| `source` | enum | auto-inferred / user-submitted / external-import | **必需** | 需求来源 |
| `usage_restriction` | enum | reference_only / none | **必需** | 使用限制（baseline 时强制 reference_only） |
| `kb_context` | string[] | KB 文件路径 | 可选 | 关联的 KB 上下文 |
| `confidence` | enum | high / medium / low | **必需** | 需求置信度（与 KB 的 float 形式不同） |
| `created` | ISO 8601 | — | **必需** | 创建时间戳 |
| `updated` | ISO 8601 | — | **必需** | 更新时间戳 |

### requirement-input.md 的 Frontmatter（简化版）

与 `requirement-doc.md` 不同，简易需求输入模板的 frontmatter 较为精简：

```yaml
---
type: <enum>                    # feature | bugfix | enhancement | refactor | chore
priority: <enum>                # high | medium | low
source: <enum>                  # cli | api | ide | webhook | manual
related_modules: <string>       # 关联模块（逗号分隔字符串，非数组）
created: <ISO 8601>
---
```

---

## 3. 其他模板 Frontmatter 字段

### impact-analysis.md

```yaml
---
req_id: <string>
req_title: <string>
analysis_date: <ISO 8601>
analyzed_by: <enum>            # AI | human
kb_contexts_used:
  - <string>
---
```

### verification-checklist.md

```yaml
---
req_id: <string>
req_title: <string>
verification_date: <ISO 8601>
verified_by: <enum>            # AI | human
---
```

### nfr-checklist.md

```yaml
---
title: "非功能需求强制评估清单"
version: "1.0"
purpose: <string>
---
```

### kb-manifest.yaml

```yaml
modules:
  - name: <string>
    path: <string>
    type: <string>
    keywords: <string>
    entities: <string>
    apis: <string>
    files: <integer>
    total_tokens_estimate: <integer>
    summary_tokens_estimate: <integer>
    last_analyzed: <ISO 8601>
    confidence: <string>
generated_at: <ISO 8601>
generated_by: <string>
total_modules: <integer>
```

---

## 4. NFR 检查清单的 7 个维度

由 `templates/nfr-checklist.md` 定义，每个维度包含触发条件、必须产出的内容和补充验收标准模板：

| # | 维度 | 触发条件 | 产出内容 |
|---|------|----------|----------|
| 1 | 认证/授权 (AuthN/AuthZ) | `type ∈ [feature, enhancement]` 且涉及用户操作 | 权限级别、新增权限点、认证方式 |
| 2 | 输入校验 (Input Validation) | 涉及 API 变更或数据模型变更 | 字段约束、注入防护、文件上传限制 |
| 3 | 数据保护 (Data Privacy) | 涉及用户数据或 `type ∉ [chore]` | 敏感字段识别、加密策略、日志脱敏 |
| 4 | 速率限制 (Rate Limiting) | 涉及新 API 端点 | 限流需求、粒度、阈值建议 |
| 5 | 审计日志 (Audit Logging) | `priority ∈ [high, medium]` 或涉及资金/权限/认证 | 记录类型、日志格式、禁止字段 |
| 6 | 性能 (Performance) | `type = feature` 且涉及数据查询或外部调用 | 数据量级、分页/缓存/异步、响应时间目标 |
| 7 | 可访问性 (Accessibility) | 涉及前端 UI 变更 | WCAG 级别、键盘导航、屏幕阅读器 |

### 维度的常量特征
- 维度名称和触发条件固定（硬编码在模板中）
- `must_produce` 内容根据具体需求动态分析
- 评估结果为每个维度输出 triggered 状态 + 具体评估内容
- 不适用时标注原因即可通过，不强制填表

---

## 5. 需求文档状态机

`requirement-doc.md` 的 `status` 字段定义了 9 个状态，对应 DevLoop 生命周期：

```
                         ┌─────────────┐
                         │   baseline   │ ← 代码逆向推断的基准线需求
                         └──────┬──────┘
                                │ (确认后进入流程)
                                ▼
                         ┌─────────────┐
                         │   proposed   │ ← 已提出但未设计
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │  designed   │ ← 已设计（有 Design Doc）
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │   planned   │ ← 已制定实现计划
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │   building  │ ← 实现中
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │  verifying  │ ← 验证中
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
             ┌──────▼──────┐        ┌──────▼──────┐
             │  completed  │        │   blocked   │ ← 阻塞（非终态）
             └─────────────┘        └──────┬──────┘
                                            │ (继续/取消)
                                            ▼
                                     ┌─────────────┐
                                     │ superseded  │ ← 被其他需求替换
                                     └─────────────┘
```

### 状态转换规则

| 从 | 到 | 条件 |
|----|----|------|
| baseline | proposed | 基线需求被确认为正式需求 |
| proposed | designed | Design Doc 评审通过 |
| designed | planned | 实现计划创建完成 |
| planned | building | Build 阶段开始 |
| building | verifying | 实现完成，进入验证 |
| verifying | completed | 所有门禁通过 |
| verifying | building | 验证失败，返回修复 |
| 任意 | blocked | 遇到阻塞因素 |
| blocked | 原状态 | 阻塞解除 |
| 任意 | superseded | 被更高优先级需求替代 |

> 注意：`requirement-doc.md` 中状态枚举类型包含 9 个值（含 superseded），相比模板注释中列出的（不含 superseded）更为完整。模板注释中的列表可能不完整，以 `{{status}}` 条件判断中的值域为准。
