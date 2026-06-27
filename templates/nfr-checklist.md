---
title: 非功能需求强制评估清单
version: "1.0"
purpose: |
  在阶段二（需求摄入）的影响分析之后强制执行。
  每个维度根据需求的 type/priority/涉及模块判定触发条件。
  对触发的维度，生成具体的补充验收标准并注入需求文档。
---

# 非功能需求强制评估清单 (NFR Checklist)

> **强制步骤**: 阶段二在影响分析完成后必须加载此清单，遍历所有 7 个维度。
> **执行时机**: 影响分析完成后，需求文档生成前。
> **输出**: 影响分析报告的「非功能需求评估」章节 + 需求文档验收标准中的对应 checklist 项。

## 评估维度

### 1. 认证/授权 (Authentication & Authorization)

- **触发条件**: `type ∈ [feature, enhancement]` 且涉及用户操作
- **必须产出的内容**:
  - 需要的权限级别（匿名/登录/角色/管理员）
  - 是否需要新增权限点
  - 认证方式（Session Token / JWT / OAuth2 / API Key）
- **补充验收标准模板**:
  - [ ] {{ auth_check_item }}

### 2. 输入校验 (Input Validation)

- **触发条件**: 涉及 API 变更或数据模型变更
- **必须产出的内容**:
  - 每个新字段的类型/范围/长度约束
  - SQL 注入/XSS/命令注入防护要求
  - 文件上传的类型/大小限制（如适用）
- **补充验收标准模板**:
  - [ ] {{ validation_check_item }}

### 3. 数据保护 (Data Privacy & Protection)

- **触发条件**: 涉及用户数据或 `type ∉ [chore]`
- **必须产出的内容**:
  - 敏感字段识别（PII / 密码 / Token / 密钥）
  - 存储加密策略（传输中 TLS、静态存储 AES-256-GCM）
  - 日志脱敏策略（哪些字段禁止出现在日志中）
- **补充验收标准模板**:
  - [ ] {{ privacy_check_item }}

### 4. 速率限制 (Rate Limiting)

- **触发条件**: 涉及新 API 端点
- **必须产出的内容**:
  - 是否需要限流
  - 限流粒度（用户级 / IP级 / 端点级）
  - 限流阈值建议
- **补充验收标准模板**:
  - [ ] {{ rate_limit_check_item }}

### 5. 审计日志 (Audit Logging)

- **触发条件**: `priority ∈ [high, medium]` 或涉及资金/权限/认证操作
- **必须产出的内容**:
  - 需要记录的操作类型（登录 / 权限变更 / 数据修改 / 资金操作）
  - 日志格式要求（含时间戳 / 用户ID / 操作类型 / 结果）
  - 日志中禁止包含的字段（密码 / Token 明文 / 完整 PII）
- **补充验收标准模板**:
  - [ ] {{ audit_check_item }}

### 6. 性能 (Performance)

- **触发条件**: `type = feature` 且涉及数据查询或外部调用
- **必须产出的内容**:
  - 预期数据量级（日活用户 / 数据行数 / QPS）
  - 是否需要分页 / 缓存 / 异步处理
  - 关键路径的响应时间目标
- **补充验收标准模板**:
  - [ ] {{ perf_check_item }}

### 7. 可访问性 (Accessibility)

- **触发条件**: 涉及前端 UI 变更
- **必须产出的内容**:
  - WCAG 级别要求（A / AA / AAA）
  - 键盘导航支持需求
  - 屏幕阅读器兼容需求
- **补充验收标准模板**:
  - [ ] {{ a11y_check_item }}

---

## 评估流程

```
1. 加载此模板 (templates/nfr-checklist.md)
2. 根据需求的 type/priority/涉及模块，判定每个维度的触发条件
3. 对触发的每个维度:
   a. 分析当前需求描述中是否已覆盖该约束
   b. 如未覆盖 → 生成具体的补充验收标准，追加到需求文档
   c. 如已覆盖 → 标记 ✅ 并引用原文位置
   d. 如不适用 → 标注「不适用」及原因（一句即可）
4. 对未触发的维度:
   a. 标注「不适用」及原因
5. 汇总为「非功能需求评估」章节，追加到影响分析报告末尾
6. 将每个触发的维度对应的验收标准注入需求文档的「验收标准」部分
```

## 评估产出格式

```markdown
## 非功能需求评估

> 根据 `templates/nfr-checklist.md` 强制执行，评估时间: {{ assessment_timestamp }}

### 认证/授权
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}

### 输入校验
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}

### 数据保护
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}

### 速率限制
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}

### 审计日志
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}

### 性能
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}

### 可访问性
- 触发: {{ triggered }} ({{ trigger_reason }})
- {{ assessment_content }}
```

## 不适用声明规则

如果某个维度确实不适用，标注原因即可通过——不要求无意义的填表。

示例：
- `NFR-可访问性: 不适用（无前端 UI 变更）`
- `NFR-速率限制: 不适用（无新增 API 端点）`
- `NFR-输入校验: 不适用（chore，无 API 变更）`

---
<!-- Variables:
  assessment_timestamp — ISO 8601 timestamp of assessment
  For each dimension:
    triggered — ✅ 触发 | ❌ 未触发
    trigger_reason — why this dimension was triggered or not
    assessment_content — specific assessment and supplementary acceptance criteria
-->
