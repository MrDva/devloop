---
name: devloop-intake
description: "DevLoop 阶段二：需求摄入 — 解析输入→查KB索引→影响分析→NFR强制评估→生成需求文档→门禁→等确认→提交。"
---

# DevLoop Intake — 阶段二：需求摄入

接收外部需求输入，结合知识库（KB）生成结构化需求文档和 OpenSpec Change。
强制执行非功能需求（NFR）评估，基线需求标记 `usage_restriction: reference_only`。

## 触发条件

1. 用户提交新需求（自然语言/用户故事/Bug报告/技术规格/PR Comment）
2. `/devloop intake <需求描述>` 命令
3. `devloop-loop` 路由 `proposed` 状态的需求到此阶段（从完善开始）
4. 外部系统通过 webhook/API 提交需求

## 流程总览

```
Step 1: 输入解析        → (编排层 LLM)                         → 标准化需求结构
Step 2: KB 上下文加载   → codegraph_explore (MCP) + .manifest.yaml  → Top-N 匹配
Step 3: 影响分析        → brainstorming                          → 影响分析报告
Step 4: NFR 强制评估     → templates/nfr-checklist.md (编排层)    → NFR 评估 + 补充验收标准
Step 5: 需求文档生成     → openspec-propose + requirement-doc.md  → requirements/in-progress/ + openspec/
Step 6: 门禁校验         → devloop-guard check stage-2           → G2.1 - G2.5
Step 7: 确认队列         → confirmation-queue + 人工确认          → 确认/驳回/修改
Step 8: Git 提交         → git commit                             → [devloop] stage:2 action:intake req:<id>
```

每个 Step 结束后调用 `comet-state checkpoint` 保存断点，支持中断恢复。

---

## Step 1: 输入解析

**目标**: 将任意格式的需求输入标准化为统一结构。

### 支持的输入格式

| 格式 | 来源 | 示例 |
|------|------|------|
| 自然语言 | CLI / IDE / API | "添加 OAuth2 第三方登录" |
| 用户故事模板 | 项目管理系统 | "As a user, I want to..." |
| Bug 报告 | Issue Tracker | "登录页面在 Safari 中无法加载" |
| 技术规格 | 技术文档 | 接口定义、数据格式要求 |
| PR/Comment | Git | PR Review 中提出的改进需求 |

### 标准化结构

使用 `templates/requirement-input.md` 作为填充目标，LLM 从输入中提取并填充：

```yaml
type: feature | bugfix | enhancement | refactor | chore  # 必填，枚举值
priority: high | medium | low                               # 必填
source: cli | api | ide | webhook | manual                 # 自动检测
related_modules: []                                         # 自动匹配或手动指定
```

**必填字段校验**:
- `title` — 从输入中提炼的简短标题
- `description` — 完整功能描述，保留原始输入语义
- `type` — 必须是有效枚举值之一
- 至少 1 条 `acceptance_criteria`

若输入无法满足必填字段，返回具体缺失提示，要求用户补充。

**门禁 G2.1** — 输入有效性:
- `title` 非空且 ≥ 5 字符
- `description` 非空且 ≥ 20 字符
- `type` ∈ `[feature, bugfix, enhancement, refactor, chore]`
- `priority` ∈ `[high, medium, low]`

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 2 2.1 "Input parsed: <title> (type=<type>, priority=<priority>)"
```

---

## Step 2: KB 上下文加载（索引式按需加载）

**目标**: 从需求描述中提取关键词，通过 `.manifest.yaml` 索引匹配相关 KB 条目，
按 token 预算逐级加载。

### 2.1 关键词提取

从标准化后的需求结构（`title` + `description` + `related_modules`）中提取关键词：
- **模块名**: `related_modules` 直接指定，或从 `description` 中匹配 `entities` 字段
- **功能词**: 从 `title`/`description` 中提取动词+名词组合（如 `oauth`, `login`, `token`, `auth`）
- **实体名**: 匹配 `.manifest.yaml` 中各模块的 `entities` 字段

### 2.2 索引匹配

读取 `knowledge-base/.manifest.yaml`，对每个模块的 `keywords` 字段做交集匹配：

```
relevance_score = |input_keywords ∩ module.keywords| / |input_keywords|
```

按 `relevance_score` 降序排列，取 Top-N（N 由 token 预算决定）。

### 2.3 分级加载策略

| 优先级 | 加载内容 | 触发条件 | Token 估算 |
|--------|---------|---------|-----------|
| **L1: 摘要层** | 匹配模块的 `overview.md`（仅 frontmatter + 首段） | 始终加载 | ~500/module |
| **L2: API 层** | 匹配模块的 `api.md` | 需求涉及 API 变更 | ~1000/module |
| **L3: 完整层** | 匹配模块的全部 5 个 KB 文件 | 高相关度（score >= 0.7）且 token 预算充裕 | ~3000/module |
| **G: 全局层** | `architecture/overview.md` + `apis/internal.md`（摘要） | 始终加载 | ~1000 |

### Token 预算控制

- 总 KB 上下文预算: **≤ 8000 tokens**
- 超出预算时从 L3 → L2 → L1 逐级降级
- 仅加载高相关度（score >= 0.5）的模块——score < 0.5 的模块不加载，防止无关上下文污染

**门禁 G2.2** — 上下文相关性（软门禁，仅 WARN）:
- 至少匹配到 1 个相关模块（score >= 0.5）
- 若匹配数为 0: 标记 `confidence: low`，主会话显式提示用户确认，但仍继续处理（可能为新模块）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 2 2.2 "KB loaded: N modules (scores: <list>), M tokens"
```

---

## Step 3: 影响分析

**目标**: 结合 KB 上下文，分析需求的影响范围。

调用 `brainstorming` Skill，输入包括：
- 标准化需求结构（Step 1 产出）
- 匹配的 KB 模块条目（Step 2 产出）
- `templates/impact-analysis.md` 模板

**brainstorming 输入 prompt 结构**:
```
你正在分析以下需求的影响范围：

[需求]
标题: {title}
类型: {type}
描述: {description}

[KB 上下文]
{loaded_kb_entries}

请按 `templates/impact-analysis.md` 模板生成影响分析报告，必须包含：
1. 涉及模块（含影响程度: major/minor/cosmetic）
2. API 变更（含兼容性评估）
3. 数据模型变更（含迁移需求）
4. 风险评估（含缓解措施）
```

**必须产出的章节**:
- **涉及模块**: 每个受影响模块的 `name`, `impact_level`, `change_description`, `estimated_files`
- **API 变更**: 若无变更，标明「✅ 无 API 变更」
- **数据模型变更**: 若无变更，标明「✅ 无数据模型变更」
- **风险评估**: 若涉及模块 > 3，必须包含；否则可选

**门禁 G2.3** — 影响分析完整性（mixed: hard + soft）:
- **hard**: 必须包含「涉及模块」「API 变更」「数据模型变更」三个章节
- **hard**: 涉及模块 > 3 时必须包含「风险评估」章节
- **soft**: LLM 评估各章节分析的深度和合理性

**基线需求用途限制**:
若 Step 2 加载的 KB 中包含基线需求（`status: baseline`, `usage_restriction: reference_only`）：
- 影响分析可将其作为「背景理解」引用
- 在影响分析报告中标注: `参考基线需求: {REQ-IDs}（仅供参考，不作为新设计约束）`
- **严格禁止**将基线需求作为影响分析的唯一依据或否决因素

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 2 2.3 "Impact analysis complete: N modules, M API changes"
```

---

## Step 4: 非功能需求强制评估 (NFR) 🆕

> **设计动机**: 安全/性能/可访问性/合规等非功能需求无法从代码逆向推断。
> KB 对这些维度天然沉默，下游 LLM 会将沉默解读为「不需要」。
> 此步骤在所有需求摄入时强制执行，不依赖 KB 内容，不可跳过。

**输入**:
- 标准化需求结构（`type`, `priority`, `related_modules`）
- 影响分析报告（Step 3 产出）
- `templates/nfr-checklist.md`

**执行流程**:

```
1. 加载 templates/nfr-checklist.md 定义的 7 个评估维度
2. 根据需求特征，判定每个维度的触发条件
3. 对触发的每个维度:
   a. 分析当前需求描述中是否已覆盖该约束
   b. 如未覆盖 → 生成具体的补充验收标准（每条以 - [ ] 开头）
   c. 如已覆盖 → 标记 ✅ 并引用原文位置
4. 对未触发的维度:
   a. 标注「不适用」及原因（一句即可）
5. 汇总为「非功能需求评估」章节，追加到影响分析报告末尾
6. 将所有触发维度的补充验收标准注入需求文档的「验收标准」部分
```

### 7 维度触发条件速查

| # | 维度 | 触发条件 |
|---|------|---------|
| 1 | **认证/授权** (AuthN/Z) | `type ∈ [feature, enhancement]` 且涉及用户操作 |
| 2 | **输入校验** (Validation) | 涉及 API 变更或数据模型变更 |
| 3 | **数据保护** (Privacy) | 涉及用户数据或 `type ∉ [chore]` |
| 4 | **速率限制** (Rate Limit) | 涉及新 API 端点 |
| 5 | **审计日志** (Audit) | `priority ∈ [high, medium]` 或涉及资金/权限/认证操作 |
| 6 | **性能** (Performance) | `type = feature` 且涉及数据查询或外部调用 |
| 7 | **可访问性** (A11y) | 涉及前端 UI 变更 |

详见 `templates/nfr-checklist.md` 中每个维度的触发条件、必须产出的内容和补充验收标准模板。

### 评估产出格式

追加到影响分析报告末尾：

```markdown
## 非功能需求评估

> 根据 `templates/nfr-checklist.md` 强制执行，评估时间: {ISO-8601}

### 认证/授权
- 触发: ✅ | ❌ ({触发原因或不适用的原因})
- 评估: {具体分析}
- 补充验收标准:
  - [ ] {具体可验证的补充验收项}

### 输入校验
- 触发: ✅ | ❌ ({触发原因或不适用的原因})
- ...

{其余维度同格式}
```

### 不适用声明规则

若维度不适用，标注原因即可通过——不要求无意义填表：
- `NFR-可访问性: 不适用（无前端 UI 变更）`
- `NFR-速率限制: 不适用（无新增 API 端点）`
- `NFR-输入校验: 不适用（chore，无 API 变更）`

**门禁 G2.3-NFR** — NFR 评估完整性（追加到 G2.3 hard checks）:
- 影响分析报告必须包含「非功能需求评估」章节
- 所有触发条件满足的维度，必须在需求文档的「验收标准」中有对应的 checklist 项
- 未触发的维度必须注明「不适用」及原因
- 失败动作: `block_and_retry`（触发补充评估）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 2 2.4 "NFR assessment complete: {M}/{7} dimensions triggered, {N} supplementary acceptance criteria"
```

---

## Step 5: 需求文档生成

**目标**: 生成统一结构的需求文档和 OpenSpec Change。

### 5.1 生成需求文档

使用 `templates/requirement-doc.md` 模板，填充以下变量：

```yaml
# 必填字段
id: REQ-{NNN}                    # 自动分配，从已有需求递增
title: {从 Step 1}
type: {从 Step 1}
priority: {从 Step 1}
status: proposed                 # 初始状态
module: {primary_module}
source: user-submitted           # 区别于基线需求的 auto-inferred
usage_restriction: none          # 用户需求无用途限制

# 上下文关联
kb_context:                      # Step 2 加载的 KB 路径列表
  - knowledge-base/modules/{module}/overview.md
  - knowledge-base/architecture/overview.md

# 影响分析产物
affected_modules:                # 从 Step 3
api_changes:                     # 从 Step 3
data_model_changes:              # 从 Step 3
impact_analysis:                 # Step 3 完整报告引用
risks:                           # 从 Step 3

# NFR 评估产物
nfr_assessment:                  # Step 4 完整评估
acceptance_criteria:             # 原始验收标准 + NFR 补充验收项（合并去重）
```

**产出路径**: `requirements/in-progress/REQ-{NNN}-{slug}.md`

### 5.2 创建 OpenSpec Change

调用 `openspec-propose` Skill，输入为已填充的需求文档，产出：

```
openspec/changes/<change-name>/
├── .comet.yaml           # 提案元数据（含需求 ID）
├── proposal.md           # 需求提案（对应需求文档的「功能描述」章节）
├── tasks.md              # 初始任务清单（空骨架，后续阶段填充）
└── specs/
    └── <spec-delta>.md   # Delta spec（初始为空，后续阶段增量填充）
```

### 5.3 确认级别判定

根据需求特征自动判定确认级别：

| 级别 | 触发条件 | 确认流程 |
|------|---------|---------|
| **Level 1** (低保障) | `type=bugfix` 且 `priority=low` 且涉及模块 ≤ 1 | 零确认直到阶段四验证 |
| **Level 2** (标准保障) | 不满足 Level 1 和 Level 3 的条件 | Design 后确认，Plan 自动继续 |
| **Level 3** (高保障) | `priority=high` 或 `type=feature` 或涉及模块 ≥ 3 | Design 确认 + Plan 确认，两轮人工评审 |

**默认级别**: Level 3（除非明确满足更低级别的条件）。

**门禁 G2.4** — 需求文档结构:
- frontmatter 包含所有必填字段
- `kb_context` 引用有效（文件存在）且 ≥ 2 个
- `acceptance_criteria` 中包含了所有 NFR 评估触发的补充验收项
- 需求文档内容与 `openspec/changes/<name>/proposal.md` 一致
- 无与已有需求的 ID 冲突

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 2 2.5 "Requirement doc: REQ-{NNN} (status=proposed, level={L})"
.comet/scripts/comet-state set active_requirement.id REQ-{NNN}
```

---

## Step 6: 门禁校验

运行阶段二全部门禁：

```bash
.comet/scripts/devloop-guard check stage-2
```

必须看到 `ALL CHECKS PASSED` 或确认失败项可接受后方可继续。

**门禁 G2.5** — 阶段二总门禁（汇总判定）：
- [ ] G2.1: 输入有效性通过
- [ ] G2.2: KB 上下文相关性通过（或已标记 low confidence 且用户确认）
- [ ] G2.3: 影响分析完整（hard checks 全部通过，soft checks 已评估）
- [ ] G2.3-NFR: 非功能需求评估完整（所有触发维度已产出验收标准）
- [ ] G2.4: 需求文档结构正确（含 NFR 补充验收项，kb_context 引用有效）
- [ ] 需求文档 frontmatter `status` 为 `proposed`
- [ ] 无与已有需求的 ID 冲突

**软门禁处理**: 若 guard 输出 `SOFT_GATES_PENDING`，由编排层委托 LLM agent 逐项评估软门禁（G2.2, G2.3-soft），结果写入 `.comet/gate-results/soft-gate-g2.<n>.json`。软门禁只能 WARN 不阻断。

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G2.5 <pass|fail> "Stage-2 gates complete"
```

---

## Step 7: 确认队列

根据 Step 5.3 判定的确认级别，将需求加入确认队列：

```bash
# Level 1: 不加入确认队列，直接进入下一步
# Level 2/3: 加入确认队列
.comet/scripts/confirmation-queue list
.comet/scripts/comet-state add-pending intake_proposed REQ-{NNN} "Stage 2 complete: level {L}, waiting confirmation"
```

**确认流程**:
1. 显示需求摘要（title, type, priority, affected_modules, NFR 评估摘要）
2. 用户选择: `confirm` (继续) | `reject` (驳回) | `skip` (暂缓) | `modify` (返回修改)
3. `confirm` → 进入 Step 8
4. `reject` → 更新 status 为 `rejected`，不提交
5. `skip` → 标记 `pending_confirmations`，等待下一轮 loop
6. `modify` → 返回用户修改指令，重新执行 Step 1-6

---

## Step 8: Git 提交

```bash
git add requirements/in-progress/REQ-{NNN}-{slug}.md
git add openspec/changes/<change-name>/
git add .comet/loop-state.yaml
git commit -m "[devloop] stage:2 action:intake req:REQ-{NNN}

- type: {type}
- priority: {priority}
- confirmation_level: {L}
- affected_modules: {N}
- nfr_dimensions_triggered: {M}/7
- kb_match_score: {top_score}

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**状态保存**:
```bash
.comet/scripts/comet-state complete-step 2 all
```

---

## 确认级别速查

| 维度 | Level 1 (低保障) | Level 2 (标准保障) | Level 3 (高保障) |
|------|-----------------|-------------------|-----------------|
| 触发条件 | `bugfix` + `low` + ≤1 模块 | 默认 | `high` 或 `feature` 或 ≥3 模块 |
| Design 确认 | ❌ 不暂停 | ✅ 暂停 | ✅ 暂停 |
| Plan 确认 | ❌ 不暂停 | ❌ 自动继续 | ✅ 暂停 |
| NFR 评估 | ✅ 仍然强制 | ✅ 仍然强制 | ✅ 仍然强制 |
| 代码审查 | 轻量（单审查者） | 标准（双审查者） | 严格（双审查者 + 安全审查） |

---

## 复用清单

| 调用 | 用途 |
|------|------|
| `codegraph_explore` (MCP) | 按关键词查询 KB 索引 |
| `brainstorming` | 影响分析（提供 KB 上下文作为输入） |
| `openspec-propose` | 创建 OpenSpec Change（proposal.md + .comet.yaml） |
| `comet-state` (Phase 0) | checkpoint + 状态管理 |
| `devloop-guard` (Phase 0) | 门禁 G2.1-G2.5 |
| `confirmation-queue` (Phase 0) | 确认队列管理 |

### 编排层新建逻辑

| 逻辑 | 说明 |
|------|------|
| 输入解析 | 任意格式 → 标准化结构（title, type, priority, description, acceptance_criteria） |
| 索引式 KB 加载 | 关键词提取 → `.manifest.yaml` → Top-N 匹配 → 摘要降级 |
| 确认级别判定 | type/priority/affected_modules → Level 1/2/3 |
| NFR 强制评估 | 加载 `nfr-checklist.md` → 遍历 7 维度 → 补充验收标准 → 注入需求文档 |
| 需求文档生成 | 使用 `requirement-doc.md` 模板，LLM 填充变量 |
| 基线需求用途限制 | `usage_restriction: reference_only` 标注 + 禁止作为新设计约束 |

---

## 防御原则

1. **NFR 强制评估不可跳过**: 阶段二在影响分析完成后，必须加载 `templates/nfr-checklist.md` 并遍历全部 7 个维度。每个维度给出「触发/不适用」判定和原因，不可跳过任何维度
2. **基线需求用途限制**: 任何 `status: baseline` + `usage_restriction: reference_only` 的需求仅供背景参考。影响分析可引用做背景理解，但严格禁止将其作为新功能设计的强制约束或否决依据
3. **索引式 KB 加载**: 必须通过 `.manifest.yaml` 的关键词匹配加载 KB 条目，不得全文加载所有 KB 文件。Token 预算超出时逐级降级（L3→L2→L1），防止上下文污染
4. **来源可追溯**: 需求文档记录 `kb_context`（加载了哪些 KB 条目）、`source`（输入来源）、`nfr_assessment`（NFR 评估结果）
5. **确认分级**: 不跳过 Level 2/3 的确认暂停点。Level 1 的快速通道仅适用于低风险 bugfix
6. **增量提交**: 需求文档生成后立即 git commit，不积攒多个需求的变更
