---
module: kb-templates
source: auto-generated
confidence: 1.0
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - templates/
---

# kb-templates — 业务逻辑

## 1. 模板变量填充流程

模板是"半成品"文档，由 LLM 或编排引擎读取后填充变量生成最终产出物。核心流程如下：

### 基本流程

```
1. LLM/Skill 读取模板文件（.md 或 .yaml）
2. LLM 分析当前上下文（代码库、需求输入、已有 KB 条目等）
3. LLM 解析模板中的 {{ }} 占位符和 {{#each}} {{#if}} 结构
4. LLM 根据上下文推断每个变量的具体值
5. 填充变量，生成完整的 Markdown/YAML 文档
6. 写入目标文件（如 knowledge-base/modules/<name>/ 或 requirements/）
```

### 变量填充策略

| 变量类型 | 填充策略 | 示例 |
|----------|----------|------|
| 元数据字段 | 直接赋值 | `module_name: "auth-service"` |
| 枚举字段 | 从枚举值中选择 | `status: "proposed"` |
| 列表（#each） | 遍历数据源构建数组 | `core_functions: [{name, summary}, ...]` |
| 条件块（#if） | 根据逻辑条件决定渲染 | status=="baseline" 时渲染警告框 |
| 后填充字段 | 留空，由后续阶段填充 | `nfr_assessment: ""` |

### 渲染引擎假设

- 模板使用 Handlebars 风格语法（`{{ }}`、`{{#each}}`、`{{#if}}`）
- 循环结构 `{{#each list}}{{this.property}}{{/each}}` 支持嵌套对象属性
- 条件判断 `{{#if condition}}...{{/if}}` 支持字符串比较
- 实际执行时不一定使用 Handlebars 库——LLM 可直接理解和填充

### 阶段对应关系

| DevLoop 阶段 | 使用的模板 | 填充方式 |
|-------------|-----------|----------|
| Stage 1 (Reverse) | kb-module-* (5个) | codegraph 分析代码后自动生成 |
| Stage 2 (Requirements Input) | requirement-input.md | 从 CLI/API/IDE 等输入源接收 |
| Stage 2 (Requirements Doc) | requirement-doc.md | 由 input 扩展 + impact-analysis + nfr 填充 |
| Stage 2 (Impact Analysis) | impact-analysis.md | KB 查询 + 依赖分析后生成 |
| Stage 2 (NFR Assessment) | nfr-checklist.md | 按 7 维度遍历评估 |
| Stage 4 (Verification) | verification-checklist.md | 测试 + 审查后填充 |
| 全局（Module Index） | kb-manifest.yaml | 增量更新模块索引 |

---

## 2. 基线需求用途限制逻辑

由 `requirement-doc.md` 的 `usage_restriction` 字段控制，核心规则：

### usage_restriction 定义

| 值 | 含义 | 适用场景 |
|----|------|----------|
| `reference_only` | 仅供文档参考，不得作为设计约束 | baseline（代码逆向推断） |
| `none` | 无限制，正常使用 | 用户提交或外部导入的需求 |

### reference_only 的约束规则

当 `usage_restriction: reference_only` 时生效：

1. **强制渲染警告横幅**：模板通过 `{{#if status == "baseline"}}` 条件块自动渲染顶部 ⚠️ 警告框
2. **不得作为设计约束**：在 Stage 3 (Build) 设计新功能时，**不得**将 baseline 需求作为强制约束
3. **冲突时让步**：如果新需求与 baseline 需求冲突，baseline 需求自动让路
4. **限在逆向分析中使用**：baseline 需求来源于代码逆向工程，反映的是"代码当前做了什么"，而非"原本设计意图是什么"
5. **仅提供背景参考**：可用于理解当前代码的行为和意图，但不约束后续设计决策

### 设计原理

```
代码实现  ≠  设计意图
   ↑                ↑
 baseline         proposed/用户需求
(reference_only)  (none)
```

baseline 需求是对现有代码行为的逆向描述，可能存在历史债务、已知缺陷或尚未清理的中间状态，因此不应作为新功能设计的硬性约束。

---

## 3. NFR 评估触发逻辑

NFR 检查清单定义 7 个维度，每个维度有独立的触发条件。评估流程如下：

### 整体流程

```
1. 影响分析完成后 → 强制加载 nfr-checklist.md
2. 获取需求文档中的 type、priority、primary_module 字段
3. 遍历 7 个维度，逐一判断触发条件
4. 触发的维度 → 分析覆盖程度 → 生成补充验收标准
5. 未触发的维度 → 标注"不适用"及原因
6. 汇总为"非功能需求评估"章节 → 追加到影响分析报告 + 注入需求文档验收标准
```

### 各维度触发判定表

| 维度 | 逻辑条件 | 不适用场景示例 |
|------|----------|---------------|
| 认证/授权 | `type ∈ [feature, enhancement]` AND 涉及用户操作 | bugfix 不涉及用户操作 |
| 输入校验 | 涉及 API 变更 OR 数据模型变更 | chore，无 API/数据变更 |
| 数据保护 | 涉及用户数据 OR `type ∉ [chore]` | chore 类且不涉及用户数据 |
| 速率限制 | 涉及新 API 端点 | 无新增端点 |
| 审计日志 | `priority ∈ [high, medium]` OR 涉及资金/权限/认证 | low 优先级且不涉及敏感操作 |
| 性能 | `type = feature` AND 涉及数据查询/外部调用 | 纯 UI 调整无后端调用 |
| 可访问性 | 涉及前端 UI 变更 | 纯后端变更 |

### 评估产出注入点

- **影响分析报告末尾**：「非功能需求评估」章节
- **需求文档验收标准**：追加对应 checklist 项
- **验证检查清单**：对应 NFR 检查项在 Stage 4 复用

### 不适用声明规则

不适用时一句话说明原因即可通过，不允许无意义填表：

```
NFR-可访问性: 不适用（无前端 UI 变更）
NFR-速率限制: 不适用（无新增 API 端点）
NFR-输入校验: 不适用（chore，无 API 变更）
```

---

## 4. 需求确认级别与模板的关系

### 从 requirement-input 到 requirement-doc 的升级路径

简易输入（requirement-input.md）→ 完整文档（requirement-doc.md）的转换逻辑：

```
requirement-input.md           requirement-doc.md
┌──────────────────┐           ┌──────────────────────┐
│ type             │──────►    │ type                 │
│ priority         │──────►    │ priority             │
│ title            │──────►    │ title                │
│ description      │──────►    │ description          │
│ acceptance_criteria│─────►   │ acceptance_criteria  │
│ background       │──────►    │ background           │
│ ─────            │           │ req_id (自动分配)     │
│ ─────            │           │ status (默认 proposed)│
│ ─────            │           │ source (auto-inferred)│
│ ─────            │           │ usage_restriction     │
│ ─────            │           │ kb_contexts (查询填充)│
│ ─────            │           │ affected_modules      │
│ ─────            │           │ api_changes           │
│ ─────            │           │ data_model_changes    │
│ ─────            │           │ nfr_assessment        │
│ ─────            │           │ impact_analysis       │
│ ─────            │           │ 实现跟踪表            │
└──────────────────┘           └──────────────────────┘
```

### 确认级别判定

| 级别 | 涉及模板 | 触发条件 | 自动化程度 |
|------|----------|----------|-----------|
| L1 — 简易输入 | requirement-input.md | 任意来源的初始需求 | 完全自动化 |
| L2 — 完整文档 | requirement-doc.md | 任何进入正式流程的需求 | 自动化+人工确认 |
| L3 — 含影响分析 | requirement-doc.md + impact-analysis.md | 需要跨模块变更时自动触发 | 自动化 |
| L4 — 含 NFR 评估 | + nfr-checklist.md | 所有需求强制触发 | 自动化 |
| L5 — 含验证检查 | + verification-checklist.md | build 完成后触发 | 自动化+人工确认 |

### 模板链接关系

```
requirement-input.md
       │ (升级)
       ▼
requirement-doc.md ◄──── impact-analysis.md
       │                        │
       │                        ▼
       │               nfr-checklist.md (7维度遍历)
       │
       ▼ (Stage 4)
verification-checklist.md ◄─── 继承 acceptance_criteria + nfr_checks
```

### 向后兼容性原则

- 简易输入可以随时升级到完整文档（补充缺失字段）
- 完整文档不可降级到简易输入（丢失字段）
- 所有后填充字段（`nfr_assessment`、`impact_analysis`、门禁结果等）在填充前为空，不影响文档结构
