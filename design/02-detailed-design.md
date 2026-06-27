# 02 — 详细设计 (Detailed Design)

## 目录

1. [阶段一：逆向分析](#1-阶段一逆向分析)
2. [阶段二：需求摄入](#2-阶段二需求摄入)
3. [阶段三：代码生成](#3-阶段三代码生成)
4. [阶段四：验证与交付](#4-阶段四验证与交付)
5. [阶段五：闭环回写](#5-阶段五闭环回写)
6. [模板系统](#6-模板系统)
7. [门禁体系](#7-门禁体系)
8. [中断与恢复](#8-中断与恢复)
9. [连续性与记忆](#9-连续性与记忆)
10. [未考虑的问题](#10-未考虑的问题)
11. [Skill 通用性设计](#11-skill-通用性设计)

---

## 1. 阶段一：逆向分析

### 1.1 总编排 Skill: `devloop-reverse`

```
devloop-reverse (🆕 编排 Skill，~150 行)
├── 1.1 代码扫描        → codegraph_explore (MCP) ✅ 已有
├── 1.2 模块深度分析    → dispatching-parallel-agents ✅ 已有 (每个 agent 内部用 codegraph_node)
├── 1.3 KB 汇总整合     → (编排层逻辑) 🆕
├── 1.4 需求反向推断    → brainstorming ✅ 已有
├── 1.5 门禁校验        → devloop-guard 🆕 (T0.3)
├── 1.6 Git 提交        → git (Bash) ✅ 已有
└── 状态管理            → comet-state 🆕 (T0.2)
```

### 1.2 子步骤详解

#### 1.2.1 代码扫描

**输入**: 项目根目录  
**产出**: `knowledge-base/.manifest.yaml` — 模块清单

```yaml
# knowledge-base/.manifest.yaml
modules:
  - name: auth-service
    path: src/auth/
    type: service
    files: 23
    dependencies: [database, cache]
  - name: api-gateway
    path: src/gateway/
    type: gateway
    files: 15
    dependencies: [auth-service]
```

**Skill 调用**:
```
codegraph_explore("列出项目的所有顶层模块和组件")
```

**门禁 G1.1** — 模块覆盖率检查:
- 规则: 扫描到的模块数 >= 预期（源码目录数 × 0.8）
- 失败动作: 补充手动指定的目录，重新扫描

#### 1.2.2 模块深度分析（并行）

**输入**: 模块清单  
**产出**: 每个模块的 KB 条目

> **细粒度拆分策略见第 14.2 节**: 单模块上下文控制在 < 50K tokens，超大模块进一步拆分为子模块。

对每个模块并行调用，每个 agent 内部使用 `codegraph_node` (MCP) 获取模块符号信息：
```
# 通过 dispatching-parallel-agents 分派，每个 agent 处理一个模块
# agent 内部: codegraph_node --file src/<module>/
```

每个模块产出:
```
knowledge-base/modules/<module-name>/
├── overview.md       # 模块概述、职责
├── api.md            # 对外接口（函数签名、参数、返回值）
├── data-model.md     # 数据模型/类型定义
├── business-logic.md # 核心业务逻辑描述
└── dependencies.md   # 模块依赖关系
```

**门禁 G1.2** — 模块分析完整性:
- 规则: 每个模块目录下至少存在 `overview.md` + `api.md`
- 失败动作: 标记缺失模块，重新分析

#### 1.2.3 KB 汇总整合

**输入**: 各模块 KB 条目  
**产出**: 项目级 KB 汇总

```
knowledge-base/
├── architecture/
│   ├── overview.md        # 整体架构描述
│   ├── components.md      # 组件拓扑
│   └── data-flow.md       # 数据流描述
├── apis/
│   ├── internal.md        # 内部 API 总览
│   └── external.md        # 外部 API 总览
└── data-models/
    └── global.md          # 全局数据模型
```

**门禁 G1.3** — KB 一致性:
- 规则: 汇总文档中的引用与实际模块 KB 条目一致
- 实现: 解析所有 `[[module-name]]` 引用，验证目标存在

#### 1.2.4 需求反向推断

**输入**: 完整知识库  
**产出**: 基线需求文档

```
requirements/baseline/
├── REQ-001-auth-core.md
├── REQ-002-user-management.md
├── REQ-003-api-gateway.md
└── ...
```

每个基线需求文档结构：
```markdown
---
id: REQ-001
title: 核心认证功能
module: auth-service
status: baseline
inferred_from:
  - src/auth/login.ts
  - src/auth/token.ts
confidence: high
created: 2026-06-27
---
# REQ-001: 核心认证功能

## 推断来源
从以下代码反向推断...

## 功能描述
系统应提供用户名/密码认证能力...

## 当前实现
- 登录接口: POST /api/auth/login
- Token 管理: JWT + Refresh Token
...
```

**门禁 G1.4** — 需求覆盖率:
- 规则: 每个模块至少被 1 个需求覆盖
- 额外规则: `confidence: low` 的需求不超过 20%

#### 1.2.5 门禁汇总校验

**门禁 G1.5** — 阶段一总门禁（所有子门禁通过 + 以下规则）:
- [ ] G1.1: 模块覆盖率 >= 80%
- [ ] G1.2: 所有模块 KB 条目完整
- [ ] G1.3: KB 一致性验证通过
- [ ] G1.4: 需求覆盖率 >= 90%
- [ ] 所有产物已 `git add` 且无 merge conflict
- [ ] KB 总大小 > 0（防止空提交）

#### 1.2.6 Git 提交

```bash
git checkout -b kb/update-$(date +%Y-%m-%d)
git add knowledge-base/ requirements/baseline/
git commit -m "[devloop] stage:1 action:reverse complete:$(date)"
```

---

## 2. 阶段二：需求摄入

### 2.1 总编排 Skill: `devloop-intake`

```
devloop-intake (🆕 编排 Skill，~150 行)
├── 2.1 输入解析        → (编排层逻辑) 🆕
├── 2.2 KB 上下文加载   → codegraph_explore ✅ 已有 (MCP) + (索引逻辑) 🆕
├── 2.3 影响分析        → brainstorming ✅ 已有
├── 2.4 需求文档生成    → openspec-propose ✅ 已有
├── 2.5 门禁校验        → devloop-guard 🆕 (T0.3)
├── 2.6 Git 提交        → git (Bash) ✅ 已有
└── 状态管理            → comet-state 🆕 (T0.2)
```

### 2.2 子步骤详解

#### 2.2.1 输入解析

**支持的输入格式**:

| 格式 | 来源 | 示例 |
|------|------|------|
| 自然语言 | CLI / IDE / API | "添加 OAuth2 第三方登录" |
| 用户故事模板 | 项目管理系统 | "As a user, I want..." |
| Bug 报告 | Issue Tracker | "登录页面在 Safari 中无法加载" |
| 技术规格 | 技术文档 | 接口定义、数据格式要求 |
| PR/Comment | Git | PR Review 中提出的改进需求 |

**输入模板** (`templates/requirement-input.md`):
```markdown
---
type: feature | bugfix | enhancement | refactor
priority: high | medium | low
source: cli | api | ide | webhook
related_modules: []
---
# [标题]

## 背景
[为什么需要这个需求]

## 描述
[需求的详细描述]

## 验收标准
- [ ] 标准 1
- [ ] 标准 2

## 附加信息
[截图、链接、参考等]
```

**门禁 G2.1** — 输入有效性:
- 规则: 至少包含 `title` 和 `description`
- 规则: `type` 是有效枚举值
- 失败动作: 返回错误信息，要求补充

#### 2.2.2 KB 上下文加载

根据输入中指定的 `related_modules` 或自动匹配，加载相关知识库。

> **详细设计见第 14 节**: 索引式按需加载策略。本节为基础流程。

```
1. 关键词匹配: 输入描述 → KB 模块索引 (.manifest.yaml keywords 字段)
2. 按相关度排序，Top-N 匹配
3. 加载匹配模块的 KB 条目（优先 overview.md，token 不足时降级为摘要）
4. 加载全局 architecture + apis + data-models（摘要层）
```

**上下文预算控制**: 见第 14.3 节预算分配表。超出预算时自动降级为摘要层加载。

**门禁 G2.2** — 上下文相关性:
- 规则: 至少匹配到 1 个相关模块
- 失败动作: 标记 `confidence: low`，但仍继续（新模块可能）

#### 2.2.3 影响分析

调用 `brainstorming` Skill 进行分析：

```markdown
# 影响分析: [需求标题]

## 涉及模块
- auth-service: 需要扩展 Token 验证逻辑
- api-gateway: 需要添加 OAuth2 路由

## 数据模型变更
- User 表: 添加 oauth_provider, oauth_uid 字段

## API 变更
- 新增: POST /api/auth/oauth2/callback
- 修改: GET /api/auth/login (添加 OAuth2 选项)

## 风险
- 现有用户数据迁移
- Session 兼容性
```

**门禁 G2.3** — 影响分析完整性:
- 规则: 必须包含 `涉及模块`、`数据模型变更`、`API变更` 三部分
- 规则: 如果涉及模块 > 3，必须有 `风险` 评估

#### 2.2.4 需求文档生成

调用 `openspec-propose` 生成正式需求：

```
openspec/changes/<change-name>/
├── .comet.yaml
├── proposal.md
├── tasks.md (初始，待设计阶段完善)
└── specs/
    └── <spec-delta>.md
```

同时生成统一需求文档到 `requirements/in-progress/`:
```markdown
---
id: REQ-010
title: 添加 OAuth2 第三方登录
type: feature
priority: high
status: proposed
module: auth-service
kb_context:
  - knowledge-base/modules/auth-service/
  - knowledge-base/architecture/overview.md
created: 2026-06-27
updated: 2026-06-27
---
# REQ-010: 添加 OAuth2 第三方登录
...
```

**门禁 G2.4** — 需求文档结构:
- 规则: frontmatter 包含所有必填字段
- 规则: 需求文档与 OpenSpec proposal.md 内容一致
- 规则: `kb_context` 引用有效且不少于 2 个

#### 2.2.5 门禁汇总校验

**门禁 G2.5** — 阶段二总门禁:
- [ ] G2.1: 输入有效性通过
- [ ] G2.2: KB 上下文相关性通过（或标记 low confidence）
- [ ] G2.3: 影响分析完整
- [ ] G2.4: 需求文档结构正确
- [ ] 需求文档 frontmatter `status` 为 `proposed`
- [ ] 无与已有需求的 ID 冲突

---

## 3. 阶段三：代码生成

### 3.1 总编排 Skill: `devloop-build`

```
devloop-build (🆕 编排 Skill，~150 行)
├── 3.1 自动拉取需求    → git pull + 扫描 🆕 (编排层逻辑)
├── 3.2 需求选取        → (编排层优先级算法) 🆕
├── 3.3 方案设计        → brainstorming ✅ 已有
├── 3.4 实现计划        → writing-plans ✅ 已有
├── 3.5 执行实现        → subagent-driven-development ✅ 已有 (大需求)
│                       或 executing-plans ✅ 已有 (小需求)
│   └── (每个 task)     → test-driven-development ✅ 已有 (被 sdd 内部调用)
│   └── (每个 task)     → requesting-code-review ✅ 已有 (被 sdd 内部调用)
├── 3.6 门禁校验        → devloop-guard 🆕 (T0.3)
├── 3.7 Git WIP Commit  → git (Bash) ✅ 已有
└── 状态管理            → comet-state 🆕 (T0.2)
```

### 3.2 子步骤详解

#### 3.2.1 自动拉取需求

```bash
# 同步远程
git pull origin main

# 扫描未完成需求
# 条件: status != "completed" AND status != "blocked"
```

**产出**: 待处理需求列表（按优先级排序）

**门禁 G3.1** — 需求可构建性:
- 规则: 需求 `status` 为 `proposed` 或 `designed` 或 `planned`
- 规则: 需求的 `kb_context` 引用仍然有效（KB 未被删除）
- 失败: 跳过该需求，标记为 `blocked`

#### 3.2.2 需求选取（优先级算法）

```yaml
# 优先级计算
priority_score:
  high: 100
  medium: 50
  low: 10

dependency_score: 0 - (被阻塞需求数 × 20)  # 被更多人依赖 = 更优先

final_score: priority_score + dependency_score
```

选取 `final_score` 最高且 `status` 允许的需求。

#### 3.2.3 Comet Design

加载 `brainstorming` Skill（提供需求文档+KB上下文作为输入）:

**步骤**:
1. 加载需求文档 + KB 上下文
2. 加载 brainstorming（如需要中等以上变更）
3. 创建设计文档: `design.md` + delta spec
4. 运行 guard: `devloop-guard <name> design`

**产物**:
```
openspec/changes/<change-name>/
├── design.md           # 设计文档
├── specs/
│   └── <name>/spec.md  # Delta spec
└── .comet.yaml         # phase: design
```

**门禁 G3.2** — Design 阶段门禁（继承 Comet design guard）:
- [ ] Design Doc 存在且非空
- [ ] Delta spec 定义了变更范围
- [ ] 设计不违反 KB 中记录的架构约束
- [ ] `devloop-guard` 返回 ALL CHECKS PASSED

**人工确认点 🔴**: Design 完成后暂停，等待用户确认设计方案。

#### 3.2.4 Comet Plan

加载 `writing-plans` Skill:

**步骤**:
1. 基于 Design Doc 创建实现计划
2. 任务拆分（每个 task < 200 行变更）
3. 创建 `plan.md` + 更新 `tasks.md`

**产物**:
```
openspec/changes/<change-name>/
├── plan.md             # 实现计划
├── tasks.md            # 任务列表（含 checkbox）
└── .comet.yaml         # phase: build
```

**门禁 G3.3** — Plan 阶段门禁:
- [ ] 所有 task 有唯一编号
- [ ] 每个 task 关联到具体的 spec delta
- [ ] 无循环依赖
- [ ] 预估总工作量在可接受范围

**人工确认点 🔴**: Plan 完成后暂停，用户选择继续/暂停/调整。

#### 3.2.5 Comet Build

根据 `build_mode` 选择：

**模式 A: `executing-plans`** (主会话执行)
- 适合小型需求（< 5 tasks）
- Skill: `executing-plans` + `test-driven-development`

**模式 B: `subagent-driven-development`** (后台子代理执行)
- 适合中大型需求
- Skill: `subagent-driven-development`
- 每个 task 由独立后台 agent 执行
- 每个 task 经过 spec compliance + code quality 双审查

**每个 Task 的内部循环**:
```
1. 实现 Task → 代码变更
2. TDD 验证 → 测试通过
3. Spec Compliance Review → 符合 spec
4. Code Quality Review → 符合标准
5. 通过 → Git commit（task 级别）
   失败 → 修复 → 回到 2（最多 3 轮）
```

**门禁 G3.4** — Build 阶段门禁:
- [ ] 所有 task checkbox 已勾选
- [ ] 所有 task 的双审查通过（subagent 模式）
- [ ] 测试覆盖率不低于变更前
- [ ] 无 linter 错误
- [ ] `devloop-guard <name> build` ALL CHECKS PASSED

---

## 4. 阶段四：验证与交付

### 4.1 总编排 Skill: `devloop-verify`

```
devloop-verify (🆕 编排 Skill，~120 行)
├── 4.1 验证纪律        → verification-before-completion ✅ 已有
├── 4.2 代码审查        → code-review ✅ 已有 (内置)
├── 4.3 问题修复        → systematic-debugging ✅ 已有 (如需要)
├── 4.4 收尾合并        → finishing-a-development-branch ✅ 已有
├── 4.5 门禁校验        → devloop-guard 🆕 (T0.3)
├── 4.6 提交+状态更新   → git (Bash) ✅ 已有
└── 状态管理            → comet-state 🆕 (T0.2)
```

### 4.2 子步骤详解

#### 4.2.1 自动化验证

加载 `verification-before-completion` Skill:

```
comet-state scale <name>        # 确定验证级别
devloop-guard check stage-4     # 运行门禁 G4.1-G4.4
```

**验证策略**: 聚焦输出正确性，不要求环境一致性。详见第 15 节。

验证内容：
- 单元测试全通过
- 集成测试全通过（如有）
- 构建成功
- Lint 零错误
- 类型检查通过
- 需求验收标准逐条覆盖（而非覆盖率数值指标）

**门禁 G4.1** — 自动化验证:
- [ ] 所有测试通过
- [ ] 构建成功
- [ ] Lint / Type Check 零错误

**失败处理**:
- 1-2 个失败 → 加载 `systematic-debugging`，定位根因，修复
- 3 次连续失败 → 暂停，用户选择：继续修复 / 接受偏差 / 放弃

#### 4.2.2 代码审查

加载 `code-review` Skill (medium 或 high):

评估维度：
- 正确性: 逻辑是否正确
- 安全性: 是否有安全漏洞
- 性能: 是否有明显性能问题
- 可维护性: 代码是否清晰
- 复用性: 是否复用了已有代码

**门禁 G4.2** — 代码审查:
- [ ] 无 Critical 级别问题
- [ ] High 级别问题数 <= 2（小型需求）或 <= 5（大型需求）
- [ ] 与 KB 记录的风格一致

#### 4.2.3 问题修复循环

```
while (存在未解决的 Critical/High 问题):
  1. systematic-debugging 定位根因
  2. 生成修复
  3. 回到 4.2.1 重新验证
  4. 问题解决 → 继续; 循环超过 3 次 → 人工决策
```

**门禁 G4.3** — 修复循环:
- 规则: 循环次数 < 3
- 规则: 每次修复后不引入新的 Critical 问题
- 超限: 暂停，等待人工决策

#### 4.2.4 最终门禁

**门禁 G4.4** — 阶段四总门禁:
- [ ] G4.1: 自动化验证全部通过
- [ ] G4.2: 代码审查达标
- [ ] `verification-before-completion` 检查通过
- [ ] 需求文档中的所有验收标准 (Acceptance Criteria) 全部满足
- [ ] KB 一致性：代码变更未破坏 KB 中的 API 契约
- [ ] Git 状态干净（无未暂存变更）

#### 4.2.5 提交与状态更新

```bash
# 1. Commit 代码变更
git add src/ openspec/changes/<name>/
git commit -m "[devloop] stage:4 action:verify req:<req-id> ✓"

# 2. 更新需求状态
# 编辑 requirements/in-progress/<req-id>.md frontmatter
# status: verifying → completed

# 3. 移动需求文档
git mv requirements/in-progress/<req-id>.md requirements/completed/<req-id>.md
git commit -m "[devloop] stage:5 action:complete req:<req-id> ✓"

# 4. 合并到 main（或提交 PR）
```

---

## 5. 阶段五：闭环回写

### 5.1 总编排 Skill: `devloop-loop`

```
devloop-loop
├── 5.1 扫描待处理需求  → git + glob
├── 5.2 队列排序        → (编排层)
├── 5.3 触发下一需求    → → 阶段二 或 阶段三
└── 5.4 监听新需求      → Cron / Webhook
```

### 5.2 子步骤详解

#### 5.2.1 扫描待处理

```bash
# 查找所有非完成需求
find requirements/in-progress/ -name "*.md" | while read f; do
  status=$(yq '.status' "$f")
  if [ "$status" != "completed" ]; then echo "$f"; fi
done
```

#### 5.2.2 队列调度

```
队列 = 按 priority_score 排序的待处理需求
WHILE (队列非空):
  取最高优先级需求
  IF 需求状态 == proposed:
    → 阶段二（从需求完善开始）
  ELIF 需求状态 in [designed, planned]:
    → 阶段三（从相应阶段继续）
  ELSE:
    跳过（异常状态，人工处理）
```

#### 5.2.3 事件监听

```yaml
# 新需求到达的触发方式
triggers:
  - type: git_webhook
    event: push
    filter: "requirements/in-progress/*.md"
  - type: cron
    schedule: "*/5 * * * *"  # 每5分钟扫描
  - type: manual
    action: "/devloop start"
  - type: api
    endpoint: "POST /devloop/intake"
```

---

## 6. 模板系统

### 6.1 模板设计原则

1. **可替换**: 所有模板通过 `templates/` 目录管理，可项目级覆盖
2. **变量化**: 使用 `{{ variable }}` 语法，运行时填充
3. **版本化**: 模板自身纳入 Git，变更可追溯
4. **可组合**: 模板之间可以相互引用 (`{{> partial }}`)

#### 6.1.1 模板引擎说明

**模板格式**: Markdown + `{{ variable }}` 占位符。模板本身是标准的 `.md` 文件，包含 `{{ }}` 包裹的变量占位符。

**模板变量**: 占位符对应需求数据、KB 数据或用户输入的字段。例如：
- `{{ title }}` — 需求标题
- `{{ priority }}` — 优先级（high/medium/low）
- `{{ req_id }}` — 需求唯一 ID
- `{{ primary_module }}` — 主影响模块名
- `{{#each kb_contexts}}...{{/each}}` — 循环遍历 KB 上下文列表

**渲染方式**: 由于所有生成均由 Claude (LLM) 完成，**不需要独立的模板引擎程序**（无需 Handlebars/Jinja）。渲染流程：
1. LLM 读取模板文件（Markdown + 占位符）
2. LLM 接收数据上下文（需求 frontmatter、KB 条目、用户输入等）
3. LLM 将占位符替换为实际值，生成最终文档
4. 输出经过门禁校验

**模板与普通 prompt 的区别**: 模板提供**结构化约束** — 强制输出必须包含特定章节、字段和格式，而非自由文本。这确保了产物的可解析性和一致性。

#### 6.1.2 模板变量来源

| 变量类别 | 来源 | 示例变量 |
|---------|------|---------|
| 需求元数据 | 用户输入 / 阶段二解析 | `title`, `type`, `priority`, `source` |
| KB 上下文 | 知识库检索结果 | `kb_contexts`, `primary_module`, `dependencies` |
| 影响分析 | brainstorming 产出 | `affected_modules`, `api_changes`, `data_model_changes` |
| 流程状态 | loop-state.yaml | `status`, `created_date`, `updated_date` |
| 验收标准 | 用户输入 + AI 推导 | `acceptance_criteria` |

### 6.2 模板清单

```
templates/
├── requirement-input.md        # 需求输入模板（外部使用）
├── requirement-doc.md          # 需求文档模板
├── kb-module-overview.md       # KB 模块概述模板
├── kb-module-api.md            # KB API 文档模板
├── kb-module-data-model.md     # KB 数据模型模板
├── kb-architecture.md          # KB 架构模板
├── impact-analysis.md          # 影响分析模板
├── design-doc.md               # 设计文档模板（覆盖 Comet 默认）
├── implementation-plan.md      # 实现计划模板
├── verification-checklist.md   # 验证检查清单模板
├── guard-config.yaml           # 门禁配置模板
└── loop-state.yaml             # 状态文件模板
```

### 6.3 模板示例

#### `requirement-doc.md`

```markdown
---
id: {{ req_id }}
title: {{ title }}
type: {{ type }}
priority: {{ priority }}
status: {{ status }}
module: {{ primary_module }}
kb_context:
{{#each kb_contexts}}
  - {{ this }}
{{/each}}
created: {{ created_date }}
updated: {{ updated_date }}
---
# {{ req_id }}: {{ title }}

## 背景
{{ background }}

## 功能描述
{{ description }}

## 涉及模块
{{#each affected_modules}}
- **{{ this.name }}**: {{ this.change_description }}
{{/each}}

## API 变更
{{#each api_changes}}
### {{ this.method }} {{ this.path }}
- 变更类型: {{ this.change_type }}
- 描述: {{ this.description }}
{{/each}}

## 数据模型变更
{{#each data_model_changes}}
### {{ this.entity }}
- 变更类型: {{ this.change_type }}
- 字段: {{ this.fields }}
{{/each}}

## 验收标准
{{#each acceptance_criteria}}
- [ ] {{ this }}
{{/each}}

## 影响分析
<!-- 由阶段二自动填充 -->

## 实现跟踪
<!-- 由阶段三/四自动更新 -->
| 阶段 | 状态 | 完成时间 |
|------|------|---------|
| Design | {{ design_status }} | {{ design_date }} |
| Plan | {{ plan_status }} | {{ plan_date }} |
| Build | {{ build_status }} | {{ build_date }} |
| Verify | {{ verify_status }} | {{ verify_date }} |
```

#### `guard-config.yaml`

```yaml
# 门禁配置 — 可项目级覆盖
gates:
  stage-1-reverse:
    - id: G1.1
      name: 模块覆盖率
      rule: "modules_analyzed / total_modules >= 0.8"
      on_fail: warn_and_continue  # 不阻断，但标记
    - id: G1.2
      name: KB 条目完整性
      rule: "all(module.has('overview.md') and module.has('api.md') for module in modules)"
      on_fail: block_and_retry
    - id: G1.3
      name: KB 内部一致性
      rule: "all_refs_valid(kb_references)"
      on_fail: block_and_retry
    - id: G1.4
      name: 需求反向覆盖率
      rule: "modules_covered_by_req / total_modules >= 0.9"
      on_fail: warn_and_continue
    - id: G1.5
      name: 阶段一总门禁
      rule: "all(G1.1.soft_pass, G1.2, G1.3, G1.4.soft_pass)"
      on_fail: block_and_retry

  stage-2-intake:
    - id: G2.1
      name: 输入有效性
      rule: "input.title and input.description"
      on_fail: reject  # 直接拒绝，不重试
    # ... more gates

# on_fail 动作说明:
#   block_and_retry: 阻断流程，自动重试（最多3次）
#   warn_and_continue: 记录警告，继续流程
#   reject: 直接拒绝，返回给调用方
#   pause_for_human: 暂停等待人工决策
```

---

## 7. 门禁体系

### 7.1 门禁分层

```
┌─────────────────────────────────────────┐
│          L3: 阶段总门禁 (Stage Gate)      │
│  阶段所有产物完备，可以进入下一阶段         │
├─────────────────────────────────────────┤
│          L2: 产物门禁 (Artifact Gate)     │
│  单项产物（需求文档/KB条目/计划）质量合格   │
├─────────────────────────────────────────┤
│          L1: 操作门禁 (Operation Gate)    │
│  单步操作（分析/生成/审查）执行正确         │
└─────────────────────────────────────────┘
```

### 7.2 门禁执行模型

```python
# 伪代码: 门禁执行引擎
def execute_gate(gate_config, context):
    for attempt in range(gate_config.max_retries):
        result = run_check(gate_config.rule, context)
        if result.passed:
            return GateResult(passed=True)
        
        action = gate_config.on_fail
        if action == "block_and_retry" and attempt < gate_config.max_retries - 1:
            context = auto_fix(gate_config, result)
            continue
        elif action == "warn_and_continue":
            log_warning(gate_config.name, result)
            return GateResult(passed=True, warnings=[result])
        elif action == "reject":
            return GateResult(passed=False, reason=result.message)
        elif action == "pause_for_human":
            human_decision = ask_user(result)
            if human_decision == "override":
                return GateResult(passed=True, overridden=True)
            else:
                return GateResult(passed=False)
    
    return GateResult(passed=False, reason="Max retries exceeded")
```

### 7.3 完整门禁矩阵

| 阶段 | 门禁ID | 名称 | 类型 | 阻断级别 | 自动修复 |
|------|--------|------|------|---------|---------|
| 1 | G1.1 | 模块覆盖率 | 统计 | WARN | N |
| 1 | G1.2 | KB条目完整性 | 结构 | BLOCK | Y |
| 1 | G1.3 | KB内部一致性 | 引用 | BLOCK | Y |
| 1 | G1.4 | 需求反向覆盖率 | 统计 | WARN | N |
| 1 | G1.5 | 阶段一总门禁 | 汇总 | BLOCK | N |
| 2 | G2.1 | 输入有效性 | 格式 | REJECT | N |
| 2 | G2.2 | KB上下文相关性 | 语义 | WARN | N |
| 2 | G2.3 | 影响分析完整性 | 结构 | BLOCK | Y |
| 2 | G2.4 | 需求文档结构 | 格式 | BLOCK | Y |
| 2 | G2.5 | 阶段二总门禁 | 汇总 | BLOCK | N |
| 3 | G3.1 | 需求可构建性 | 前置 | BLOCK | N |
| 3 | G3.2 | Design阶段质量 | 过程 | BLOCK | N |
| 3 | G3.3 | Plan阶段质量 | 过程 | BLOCK | Y |
| 3 | G3.4 | Build阶段质量 | 过程 | BLOCK | Y |
| 4 | G4.1 | 自动化验证 | 测试 | BLOCK | Y |
| 4 | G4.2 | 代码审查 | 审查 | BLOCK | Y |
| 4 | G4.3 | 修复循环限制 | 过程 | PAUSE | N |
| 4 | G4.4 | 阶段四总门禁 | 汇总 | BLOCK | N |
| 5 | G5.1 | 闭环状态一致性 | 状态 | BLOCK | Y |

---

## 8. 中断与恢复

### 8.1 中断场景分类

| 场景 | 原因 | 发生阶段 | 恢复策略 |
|------|------|---------|---------|
| 会话超时 | 长时间运行 | 所有阶段 | 读取 state，从 checkpoint 继续 |
| 上下文压缩 | Token 超限 | 阶段 3 (Build) | comet-state recover |
| 人工确认等待 | 用户未响应 | 阶段 2/3/4 | 状态持久化，轮询或通知 |
| API 错误 | 模型不可用 | 所有阶段 | 指数退避重试 |
| Git 冲突 | 并发修改 | 阶段 1/4 | 标准 Git 冲突解决 |
| 门禁失败 | 质量不达标 | 所有阶段 | 根据 on_fail 策略 |

### 8.2 状态文件设计

```yaml
# .comet/loop-state.yaml
devloop:
  version: "1.0"
  updated: "2026-06-27T10:30:00Z"
  
  current_stage: 3
  current_step: 3.4  # Comet Plan
  
  active_requirement:
    id: REQ-010
    title: 添加 OAuth2 第三方登录
    change_name: add-oauth2-support
    
  checkpoint:
    stage: 3
    step: 3.4
    last_action: "plan.md created, waiting for user confirmation"
    timestamp: "2026-06-27T10:30:00Z"
    git_commit: "abc1234"
    
  completed_steps:
    stage-1:
      - step: 1.1
        status: completed
        timestamp: "2026-06-27T09:00:00Z"
      - step: 1.2
        status: completed
        timestamp: "2026-06-27T09:15:00Z"
      # ...
    stage-2:
      - step: 2.1
        status: completed
        timestamp: "2026-06-27T09:45:00Z"
      # ...
      
  gate_results:
    G1.1: { passed: true, warnings: ["module coverage: 85%"] }
    G1.2: { passed: true }
    # ...
    
  pending_confirmations:
    - type: plan_ready
      req: REQ-010
      message: "Implementation plan is ready for review"
      timestamp: "2026-06-27T10:30:00Z"
      
  errors:
    - step: 3.3
      error: "Design guard failed: missing delta spec"
      resolved: true
      resolution: "Re-ran brainstorming with explicit spec delta"
```

### 8.3 恢复流程

```
启动时:
  1. 读取 .comet/loop-state.yaml
  2. 如果存在活跃流程:
     a. 显示中断位置和上下文摘要
     b. 运行 comet-state check <name> <phase> --recover
     c. 按 Recovery action 继续
  3. 如果不存在活跃流程:
     a. 正常启动，等待指令
     
恢复优先级:
  1. 如果有 pending_confirmations → 优先处理（恢复对话上下文）
  2. 如果有 errors 未解决 → 先解决错误
  3. 否则 → 从 current_step 继续
```

### 8.4 Checkpoint 机制

每个子步骤完成后自动保存 checkpoint：

```python
def save_checkpoint(stage, step, action, context):
    state = load_state()
    state.checkpoint = {
        "stage": stage,
        "step": step,
        "last_action": action,
        "timestamp": now(),
        "git_commit": get_head_commit()
    }
    state.completed_steps[f"stage-{stage}"].append({
        "step": step,
        "status": "completed",
        "timestamp": now()
    })
    save_state(state)
    # 同时写入 memory 文件作为备份
    write_memory("devloop-checkpoint", state.checkpoint)
```

---

## 9. 连续性与记忆

### 9.1 多层记忆架构

```
┌──────────────────────────────────────────────┐
│  Layer 4: Git History (永久记忆)              │
│  所有产物、commit message、PR 讨论             │
│  跨会话、跨开发者、永不丢失                     │
├──────────────────────────────────────────────┤
│  Layer 3: Knowledge Base (长期记忆)           │
│  结构化项目知识，随代码更新而更新               │
│  AI 理解项目的核心依据                          │
├──────────────────────────────────────────────┤
│  Layer 2: Memory Files (中期记忆)              │
│  ~/.claude/projects/.../memory/*.md           │
│  跨会话的流程状态、决策记录、用户偏好            │
├──────────────────────────────────────────────┤
│  Layer 1: Session State (短期记忆)            │
│  .comet/loop-state.yaml                       │
│  当前会话的详细进度、checkpoint                 │
└──────────────────────────────────────────────┘
```

### 9.2 记忆写入时机

| 事件 | 写入层 | 内容 |
|------|--------|------|
| 完成一个子步骤 | L1 + L2 | checkpoint 更新 |
| 完成一个阶段 | L3 | Git commit |
| KB 条目更新 | L2 + L3 | 变更记录 + KB commit |
| 人工决策 | L2 | 决策记录 memory |
| 用户偏好变更 | L2 | 偏好更新 memory |
| 门禁失败 | L1 + L2 | 失败记录 + 恢复策略 |
| 流程完成 | L3 + L2 + L1 | 完成记录 + 清理 state |

### 9.3 上下文压缩后恢复

```
1. 检测压缩信号:
   - 对话上下文大幅缩短
   - 缺少之前的讨论细节
   
2. 恢复脚本:
   comet-state check <name> <phase> --recover
   
3. 脚本输出 Recovery Action:
   - "resume_from_step": 从步骤 X.Y 继续
   - "replay_missing": 重放丢失的产物生成
   - "restart_stage": 重新开始当前阶段
   
4. 快速上下文重建:
   - 读取当前 change 的 proposal.md（知道在做什么）
   - 读取 KB 相关模块（知道代码上下文）
   - 读取 loop-state.yaml（知道做到哪了）
   - 读取最近 3 个 memory 文件（知道决策历史）
```

### 9.4 Knowledge Base 保鲜机制

> **详细设计见第 16 节**: Git 驱动的 KB 保鲜定时触发。本节提供概览。

```
问题: KB 是代码的镜像，代码变更后 KB 会过时

解决方案:
1. 每次阶段四（验证交付）后自动触发 KB 增量更新
   - 只更新受影响的模块 KB 条目
   - 通过 codegraph 对比变更前后的符号差异
   
2. 定时巡检（每周 + 每月）
   - 每周: 全量扫描，生成漂移报告（不自动修改）
   - 每月: 全量扫描，自动修复高置信度漂移项
   - 详见第 16.2 节触发配置
   
3. Git 变更驱动的增量更新
   - git diff 检测自上次 KB 更新以来的代码变更
   - 映射变更文件 → 受影响模块
   - 仅对受影响模块做增量逆向
   - 详见第 16.3 节
   
4. 手动触发
   - 命令: /devloop kb-fresh --scope <module|all> --mode <report|fix>
   - 用于重大变更后或代码审查前的主动保鲜

5. 漂移阻断
   - drift_score > 0.3 的模块阻断阶段三
   - 全局平均 drift_score > 0.2 阻断所有阶段
   - 详见第 13.3 节
```

---

## 10. 未考虑的问题

### 10.1 技术类

| # | 问题 | 影响 | 状态 | 解决方案 |
|---|------|------|------|---------|
| T1 | **跨仓库需求**: 需求可能涉及多个代码仓库 | 阶段三需要跨仓库协调 | 🟡 待解决 | 引入 `repo-manifest.yaml`，定义多仓库拓扑 |
| T2 | **KB 规模膨胀**: 大项目的 KB 可能超出 AI 上下文窗口 | 阶段二/三无法加载完整上下文 | ✅ 已解决 | 见第 14 节：细粒度模块拆分 + 索引式按需加载 |
| T3 | **并发需求冲突**: 两个需求修改同一文件 | Git merge conflict | 🟡 已预留 | 见第 17 节：并发锁预留机制（仅预留设计，当前串行） |
| T4 | **非功能需求**: 性能/安全/可访问性需求难以从代码逆向 | 阶段一遗漏非功能需求 | 🟡 待解决 | 添加非功能需求检查清单模板，阶段二强制评估 |
| T5 | **测试代码的理解**: 测试代码本身也是知识来源 | 阶段一可能忽略测试中的隐含需求 | 🟡 待解决 | 将测试文件纳入逆向分析，提取用例作为验收标准 |
| T6 | **外部依赖变化**: 第三方 API 变更影响项目 | KB 中的外部 API 描述过时 | ✅ 已解决 | 见第 16 节：Git 驱动的定时保鲜 + 外部依赖扫描 |

### 10.2 流程类

| # | 问题 | 影响 | 状态 | 解决方案 |
|---|------|------|------|---------|
| P1 | **长流程疲劳**: 大需求的完整 5 阶段可能需要数小时 | 会话超时，注意力分散 | 🟡 待解决 | 支持分段执行，每阶段可独立触发 |
| P2 | **确认点积压**: 多个需求同时到达人工确认点 | 用户成为瓶颈 | ✅ 已解决 | 见第 12 节：确认策略分级 + 批量确认 |
| P3 | **回滚复杂性**: 阶段四发现严重问题需要回滚到阶段二 | 大量工作浪费 | 🟡 待解决 | 阶段三早期（Design 后）做最小可行性验证 |
| P4 | **需求变更**: 实现过程中需求被修改 | 已完成工作可能无效 | ✅ 已解决 | 见第 18 节：需求变更策略（supersedes/replaces + 版本追溯） |
| P5 | **环境一致性**: 不同开发者机器上运行结果不同 | 验证结果不可复现 | ✅ 已解决 | 见第 15 节：聚焦输出正确性（测试通过 = 正确） |

### 10.3 组织类

| # | 问题 | 影响 | 状态 | 解决方案 |
|---|------|------|------|---------|
| O1 | **角色与权限**: 谁可以接受需求？谁可以确认跳过门禁？ | 安全与合规 | 🟡 待解决 | 需求文档 frontmatter 添加 `author`/`reviewer` 字段 |
| O2 | **需求溯源**: 完成后需要知道需求来源 | 审计追踪断裂 | ✅ 已解决 | 见第 18 节：需求文档完整溯源链（source → intake → code → deploy）+ version_history |
| O3 | **知识所有权**: KB 由谁维护？AI 自动更新是否可信？ | KB 质量可能下降 | ✅ 已解决 | 见第 13 节：三级信任体系 + auto-generated/human-reviewed 标记 |
| O4 | **团队协作**: 多人同时使用 DevLoop | 状态冲突 | ✅ 已解决 | **当前范围: 单开发者**。未来扩展见第 17 节并发锁预留 |

### 10.4 质量类

| # | 问题 | 影响 | 状态 | 解决方案 |
|---|------|------|------|---------|
| Q1 | **需求推断偏差**: AI 从代码推断的需求可能不准确 | 基线需求误导后续开发 | ✅ 已解决 | 见第 13 节：confidence 分数 + inferred 标记 + 低置信度需求优先人工审核 |
| Q2 | **门禁疲劳**: 门禁过多导致开发变慢 | 团队可能绕过门禁 | 🟡 待解决 | 门禁分级（必须/建议/可选），紧急模式可降级 |
| Q3 | **测试生成质量**: AI 生成的测试可能脆弱或无效 | 虚假的安全感 | 🟡 待解决 | 测试有效性检查（mutation testing），低质量测试自动重写 |
| Q4 | **知识漂移累积**: KB 保鲜不及时，漂移逐渐扩大 | 后续需求质量下降 | ✅ 已解决 | 见第 13 节 + 第 16 节：漂移阈值自动阻断 + 定时保鲜触发 |

### 10.5 尚未覆盖的边界情况

| # | 边界情况 | 状态 |
|---|---------|------|
| B1 | **空项目启动**: 全新项目无代码可逆向 | 🟡 待解决: 初次使用时提供骨架 KB 模板，从阶段二直接开始 |
| B2 | **Legacy 代码**: 无测试、无文档的老代码 | 🟡 待解决: 阶段一降低 confidence，强制标记 tentative |
| B3 | **Mono-repo 超大项目**: 10万+文件 | ✅ 已解决: 见第 14 节细粒度拆分策略 |
| B4 | **多语言项目**: TypeScript + Python + Rust | 🟡 待解决: 阶段一按语言分治，KB 条目标记 language 字段 |
| B5 | **需求废弃**: 进行中的需求被取消 | ✅ 已解决: 见第 18 节，标记 blocked/superseded |
| B6 | **重名符号处理**: 多语言/多模块同名的函数或类 | 🟡 待解决: 通过 codegraph 的 file/line 消歧 + KB entry 命名空间隔离 |

---

## 11. Skill 通用性设计

### 11.1 设计原则

Skill 的通用性通过以下机制实现：

1. **模板注入**: Skill 从 `templates/` 读取模板，而非硬编码
2. **配置驱动**: Skill 行为由 `guard-config.yaml` 和 Skill 自身的配置文件控制
3. **接口标准化**: 每个 Skill 的输入/输出遵循标准化格式
4. **依赖声明**: Skill 声明其依赖的其他 Skill 和 MCP 工具

### 11.2 Skill 接口规范

每个 DevLoop Skill 遵循统一接口：

```yaml
# skill-meta.yaml (每个 Skill 自带)
name: devloop-reverse
version: "1.0"
description: "阶段一：代码逆向分析总编排"
category: devloop
stage: 1

inputs:
  - name: project_root
    type: path
    required: true
    default: "."
  - name: modules
    type: array
    required: false
    description: "手动指定要分析的模块列表，为空则自动扫描"

outputs:
  - name: kb_manifest
    type: file
    path: "knowledge-base/.manifest.yaml"
  - name: kb_entries
    type: directory
    path: "knowledge-base/modules/"
  - name: baseline_requirements
    type: directory
    path: "requirements/baseline/"

dependencies:
  skills:
    - brainstorming
    - dispatching-parallel-agents
  mcp_tools:
    - codegraph_explore
    - codegraph_node

templates:
  - kb-module-overview.md
  - kb-module-api.md
  - requirement-doc.md

gates:
  - G1.1
  - G1.2
  - G1.3
  - G1.4
  - G1.5
  
checkpoint_strategy: after_each_step
```

### 11.3 模板替换机制

```
Skill 加载时:
  1. 读取 Skill 声明的 templates 列表
  2. 先从项目 templates/ 目录查找
  3. 项目无 → 从 Skill 内置默认模板 fallback
  4. 模板变量由上下文填充（需求数据、KB 数据等）
  5. 渲染后的产物进入门禁校验
```

### 11.4 Skill 替换/扩展

```
场景 1: 替换阶段二的 KB 上下文加载策略
  - 创建项目级 templates/impact-analysis.md
  - 自定义 guard-config.yaml 中 G2.3 的 rule
  - Skill 自动使用项目模板和规则

场景 2: 添加新的验证门禁
  - 编辑 guard-config.yaml，在 stage-4 添加新 gate
  - 提供 rule 表达式和 on_fail 策略
  - 无需修改 Skill 代码

场景 3: 完全替换某个阶段编排
  - 创建新 Skill，实现相同接口
  - 修改 project 级 Skill registry
  - 新 Skill 接管该阶段
```

---

## 12. 确认策略分级

### 12.1 设计原则

**默认高保障**: 除非明确降级，所有需求使用最高确认级别（Level 3）。高保障是默认值，用户可主动降级但不能被系统自动降级。

### 12.2 三个确认级别

| 级别 | 名称 | 确认点 | 适用场景 | 用户操作 |
|------|------|--------|---------|---------|
| **Level 3** (默认) | 高保障 | 🔴 需求内容 + 🔴 设计方案 + 🔴 实现计划 + 🔴 验证失败决策 | `priority: high` 的 feature、涉及 >3 模块的变更、安全相关 | 每个确认点主动暂停等待 |
| **Level 2** | 标准保障 | 🟡 设计方案 + 🟡 验证失败决策（跳过需求内容确认和实现计划确认） | `priority: medium` 的 feature、重构 | Design 后确认，Plan 自动通过 |
| **Level 1** | 快速通道 | 🟢 仅验证失败决策（全自动推进至阶段四） | `priority: low` 的 bugfix、小改动、`type: chore` | 无人干预，仅异常时介入 |

### 12.3 确认级别判定

```yaml
# 确认级别判定逻辑
determine_confirmation_level:
  default: Level 3  # 默认高保障
  
  auto_downgrade_allowed:
    - condition: "type == 'bugfix' and priority == 'low'"
      level: Level 1
    - condition: "type == 'chore'"
      level: Level 1
    - condition: "priority == 'medium' and affected_modules <= 3"
      level: Level 2
  
  manual_override: true  # 用户可随时调整任何需求的确认级别
  
  cannot_downgrade:  # 以下场景禁止降级
    - "type == 'security'"
    - "affected_modules > 5"
    - "data_model_changes includes 'migration'"
```

### 12.4 确认点行为

每个确认点暂停时：
1. 展示当前产物摘要（需求文档/设计方案/实现计划）
2. 列出关键决策点和风险
3. 提供选项：✅ 确认 / ✏️ 修改 / ⏭️ 跳过（需权限）/ ❌ 放弃
4. 记录确认决策到 `loop-state.yaml` 和 memory 文件
5. 超时策略：24 小时无响应 → 标记为 `awaiting_confirmation` 状态，不自动推进

### 12.5 批量确认（优化）

当 pipeline 中存在多个同级别需求的确认点时，支持**批量确认**：
- 展示需求列表 + 每个需求的关键变更摘要
- 用户可一次性确认全部 / 逐条确认 / 选择性跳过
- 在 `loop-state.yaml` 中维护确认队列

---

## 13. 知识库防御机制

### 13.1 知识条目可信度标记

每个 KB 条目 frontmatter 必须包含：

```yaml
---
module: auth-service
source: auto-generated        # auto-generated | human-reviewed | hybrid
confidence: 0.85              # 0.0 - 1.0
last_reviewed: 2026-06-27     # 最后审核日期
reviewed_by: null             # null | human-identifier
drift_score: 0.05             # 与当前代码的漂移度 (< 0.1 = 健康)
generated_from:                # 来源追溯
  - src/auth/login.ts
  - src/auth/token.ts
---
```

### 13.2 三级信任体系

| 级别 | 标记 | 门禁行为 | 更新策略 |
|------|------|---------|---------|
| **Trusted** | `human-reviewed` + `confidence >= 0.9` + `drift_score < 0.05` | 正常通过，作为强依赖 | 仅人工更新或增量自动更新+人工审核 |
| **Tentative** | `auto-generated` + `confidence >= 0.7` | 正常通过，附加 warning | 自动更新，标记 `auto-updated` |
| **Untrusted** | `confidence < 0.7` 或 `drift_score > 0.2` | **阻断**: 必须先人工审核或触发重新逆向 | 阻止作为上下文使用，强制重建 |

### 13.3 漂移检测与自动阻断

```
漂移检测时机:
  1. 阶段一（逆向）: 对比 codegraph 符号与 KB 条目，计算 drift_score
  2. 阶段四（验证）: 代码变更后检查受影响模块的 KB 条目是否过时
  3. 定时巡检: 每周全量扫描（见第 9.4 节）

阻断规则:
  - 任意模块 drift_score > 0.3 → 阻断阶段三，强制触发该模块的增量逆向
  - 全局平均 drift_score > 0.2 → 阻断所有阶段，建议全量逆向重建
  - 低 confidence (<0.7) 的条目被 >3 个需求引用 → 阻断，要求人工审核
```

### 13.4 KB 条目交叉验证

- 阶段一生成的 KB 条目（API 签名等）与 codegraph 实时查询结果交叉比对
- 不一致时：降低 confidence，记录差异
- 阶段四完成后：用实际代码变更结果反向验证 KB 条目准确性

---

## 14. KB 上下文窗口管理

### 14.1 问题定义

大项目的 KB 总体积可能超过 LLM 上下文窗口（~200K tokens），导致阶段二/三无法加载完整上下文。需要分层加载策略。

### 14.2 逆向分析方向：细粒度模块拆分

**原则**: 阶段一不分析"整个项目"，而是分析"每个最小可独立理解的模块"。

```
拆分策略:
  1. codegraph 扫描 → 识别物理模块（目录/包）
  2. 每个模块独立分析，单模块上下文 < 50K tokens
  3. 超大模块（>100 文件）进一步拆分为子模块
  4. 每个模块生成独立的 KB 条目（相互引用而非重复内容）

拆分粒度:
  - 小型项目 (< 100 文件): 按顶层目录拆分
  - 中型项目 (100-500 文件): 按二级目录拆分，子模块分组
  - 大型项目 (500+ 文件): 按功能域拆分，多级 KB 目录树
```

**并行分析**: 各模块完全独立，使用并行 agent 同时分析（阶段一已设计并行），单个 agent 只看到自己负责的模块代码。

### 14.3 需求摄入方向：索引式按需加载

**原则**: 阶段二/三不加载完整 KB，只加载与当前需求相关的 KB 条目。

```
索引式加载流程:
  1. 需求输入 → 关键词提取（模块名、函数名、数据实体）
  2. 关键词 → KB 索引查询 (.manifest.yaml + frontmatter 字段)
  3. 匹配结果排序（按相关度）
  4. 加载 Top-N 相关条目（N 由上下文预算决定）
  5. 如果匹配条目总 token 数 > 预算 → 加载摘要层（overview.md 优先于完整 api.md）

索引结构 (knowledge-base/.manifest.yaml):
  modules:
    - name: auth-service
      keywords: [auth, login, token, oauth, session, password]
      entities: [User, Token, Session]
      apis: [POST /auth/login, POST /auth/refresh]
      files: 23
      total_tokens_estimate: 15000  # KB 条目总 token 估算
      summary_tokens_estimate: 2000  # overview.md 单独 token 估算

上下文预算分配（以 200K 窗口为例）:
  - 系统指令 + Skill 定义: ~20K
  - 需求文档: ~5K
  - KB 相关条目（索引加载）: ~80K  ← 按需加载
  - 相关源代码（codegraph）: ~50K
  - 生成产物: ~30K
  - 缓冲: ~15K
```

---

## 15. 验证策略：输出正确性聚焦

### 15.1 设计原则

**不要求环境一致性**（不同机器/会话的运行环境可能不同）。验证只关注**输出的正确性**：

| 关注 | 不关注 |
|------|--------|
| ✅ 单元测试全部通过 | ❌ 测试通过的机器环境是否一致 |
| ✅ 集成测试全部通过 | ❌ 集成测试的运行路径是否一致 |
| ✅ Lint / Type Check 零错误 | ❌ Lint 规则版本是否锁定 |
| ✅ 构建成功 | ❌ 构建工具的版本是否一致 |
| ✅ 功能行为符合 spec 定义 | ❌ 实现方式是否与某参考实现一致 |

### 15.2 测试分层

```
L1: 单元测试 — 函数/方法级别，验证逻辑正确性
L2: 集成测试 — 模块间交互，验证接口契约
L3: 端到端测试 (UAT) — 需求验收标准逐条验证

阶段四验证策略:
  - L1 + L2 必须全部通过 → 阻塞级门禁
  - L3 由验收标准驱动 → 每条 acceptance criteria 必须覆盖
  - 不要求测试覆盖率数值（避免 AI 生成低质量测试凑覆盖率）
  - 但要求"新代码必须有对应测试"（测试存在性而非覆盖率百分比）
```

---

## 16. Git 驱动的 KB 保鲜定时触发

### 16.1 设计原则

利用 Git 版本历史检测代码变更 → 触发受影响模块的 KB 增量更新。支持定时和手动触发。

### 16.2 触发方式

```yaml
# KB 保鲜触发配置
freshness_triggers:
  # 方式 1: 阶段四自动触发（每次需求完成后）
  - type: post_verify
    scope: affected_modules_only  # 只更新被需求影响到的模块 KB
    
  # 方式 2: 定时巡检（Cron）
  - type: cron
    schedule: "0 2 * * 0"        # 每周日凌晨 2:00
    scope: full_scan              # 全量扫描所有模块
    action: report_only           # 只生成漂移报告，不自动修改
    
  - type: cron
    schedule: "0 3 1 * *"        # 每月1日凌晨 3:00
    scope: full_scan
    action: auto_fix              # 自动修复置信度高的漂移项
    
  # 方式 3: 手动触发
  - type: manual
    command: "/devloop kb-fresh --scope <module|all> --mode <report|fix>"
```

### 16.3 基于 Git 的变更检测

**原理**: 使用 Git commit 祖先链而非时间戳，天然处理同日多次变更。

```bash
# 检测自上次 KB 更新以来的代码变更
LAST_KB_UPDATE=$(git log --oneline --grep="\[devloop\] stage:1" -1 --format="%H")
CHANGED_FILES=$(git diff --name-only $LAST_KB_UPDATE HEAD -- "src/")

# 映射变更文件 → 受影响模块
for file in $CHANGED_FILES; do
  module=$(echo $file | cut -d'/' -f2)  # src/<module>/...
  echo "Module $module affected by $file"
done

# 触发受影响模块的增量逆向
# devloop-reverse --scope modules=$affected_modules
```

**同日多次变更的健壮性**:
- `git diff <commit> HEAD` 捕获的是两个 commit 之间**所有**变更，与中间经过多少 commit、横跨多少天无关
- 保鲜自身也会产生 `[devloop] stage:1` commit，后续保鲜自动以最新一次为基线，不会重复检测已修复的内容
- 每周 report-only 扫描不产生 commit，不影响基线；每月 auto-fix 产生 commit 后自动推进基线

### 16.4 保鲜操作流程

```
定时触发:
  1. git diff 检测变更文件
  2. 映射到受影响模块
  3. 对每个受影响模块:
     a. codegraph 获取当前符号信息
     b. 对比 KB 条目中的 API/数据模型描述
     c. 标记差异项
  4. 生成漂移报告 (knowledge-base/.drift-report.yaml)
  
  漂移报告结构:
    module: auth-service
    drift_score: 0.12
    changes:
      - type: api_signature_changed
        kb_entry: "POST /auth/login → (username, password)"
        current: "POST /auth/login → (username, password, mfa_token)"
        auto_fixable: true
      - type: new_entity
        entity: "OAuthProvider"
        kb_entry: null  # KB 中没有记录
        auto_fixable: true
      - type: removed_function
        kb_entry: "validateLegacyToken()"
        current: null  # 代码中已删除
        auto_fixable: true

  5. auto_fixable=true 的项 → 自动更新 KB 条目
  6. auto_fixable=false 的项 → 标记，等待人工审核
  7. 更新 drift_score 和 last_updated
  8. Git commit 保鲜更新
```

---

## 17. 并发锁预留

### 17.1 当前范围

当前仅支持**单个开发者串行执行**。DevLoop 同时在处理的需求数为 1（单线程流水线）。

### 17.2 预留锁机制

当未来需要支持并发需求处理时，设计上预留以下机制：

```yaml
# .comet/loop-lock.yaml (未来启用)
lock:
  strategy: file_based           # 文件锁，最简实现
  granularity: module_level      # 按模块粒度加锁
  
  # 锁规则
  rules:
    - resource: "requirements/in-progress/"
      lock_type: exclusive_write  # 同一时间只有一个需求处于 active 构建
    - resource: "knowledge-base/modules/<name>/"
      lock_type: shared_read      # 多个需求可同时读取
    - resource: "src/<module>/"
      lock_type: exclusive_write  # 同一模块同一时间只有一个写者
  
  # 冲突检测
  conflict_detection:
    - before_build: 检查当前需求涉及的模块是否被其他活跃需求锁定
    - on_conflict: 重新排队（降低优先级 + 记录冲突原因）
```

**当前实现**: 仅保留 `loop-state.yaml` 中的 `active_requirement` 字段（单需求），不实现锁逻辑。未来需扩展时，将 `active_requirement` 改为 `active_requirements[]` 并启用 `loop-lock.yaml`。

---

## 18. 需求变更策略

### 18.1 两种变更场景

#### 场景 A: 基于已完成需求的修改

前一个需求已经做完了（`status: completed`），需要在原有基础上修改。

**策略**: 创建新需求，`supersedes` 指向原需求。

```yaml
# 新需求 frontmatter
id: REQ-015
title: OAuth2 支持 Microsoft 账号（扩展）
type: feature
priority: medium
status: draft
supersedes: REQ-010              # ← 被替代的需求
base_version: REQ-010-v1         # ← 基于哪个版本
change_type: extension           # extension | modification | replacement
```

**产物关系**:
```
requirements/completed/REQ-010.md       # 原需求（已完成，只读）
requirements/in-progress/REQ-015.md     # 新需求
                                        # frontmatter.supersedes = REQ-010
                                        # 内容引用 REQ-010 的实现作为基础
```

#### 场景 B: 完全不同的新需求覆盖旧需求

旧需求（可能还在进行中或已完成）被一个完全不同的方案取代。

**策略**: 创建新需求，`replaces` 指向旧需求。旧需求标记为 `superseded`。

```yaml
# 新需求
id: REQ-020
title: 使用 WebAuthn 替代 OAuth2（全新方案）
type: feature
priority: high
status: draft
replaces: REQ-010                # ← 完全取代
reason: "安全评审决定采用 WebAuthn 标准替代自建 OAuth2"
```

**旧需求状态变更**: `status: completed → superseded`，添加 `superseded_by: REQ-020`。

### 18.2 版本追溯

```
Git 层面:
  - 每个需求的每次变更 = 一个 commit
  - commit message 格式: [devloop] stage:2 action:intake req:REQ-015 supersedes:REQ-010
  - 可通过 git log --grep="req:REQ-010" 追溯完整变更链

需求文档层面:
  - frontmatter 维护版本链:
    version_history:
      - version: v1
        id: REQ-010
        date: 2026-06-20
        status: completed
      - version: v2
        id: REQ-015
        date: 2026-06-27
        supersedes: REQ-010-v1
        status: proposed
```

### 18.3 进行中需求的变更

```
如果 REQ-010 处于 building 阶段时收到变更:
  1. 评估已完成 task 的影响范围
  2. 如果影响 < 2 个 task → 在当前分支直接调整
  3. 如果影响 >= 2 个 task → 暂停当前需求，创建 REQ-015 (supersedes)
  4. 原 REQ-010 标记为 blocked，引用新需求
```
