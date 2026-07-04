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
devloop-reverse (🆕 编排 Skill，~200 行)
├── 1.2.1 项目感知      → codegraph_explore (MCP) ✅ 已有
├── 1.2.2 代码扫描      → codegraph_explore + codegraph_callers (MCP) ✅ 已有 (七阶段多信号聚类)
├── 1.2.3 模块深度分析  → dispatching-parallel-agents ✅ 已有 (每个 agent 内部用 codegraph_node)
├── 1.2.4 KB 汇总整合   → (编排层逻辑) 🆕
├── 1.2.5 需求反向推断  → brainstorming ✅ 已有
├── 1.2.6 门禁校验      → devloop-guard 🆕 (T0.3)
├── 1.2.7 Git 提交      → git (Bash) ✅ 已有
└── 状态管理            → comet-state 🆕 (T0.2)
```

### 1.2 子步骤详解

#### 1.2.1 项目感知（环境检测 + 策略选择）🆕

**目标**: 在聚类之前确定项目类型和技术栈，选择对应的种子策略和分类体系。

**输入**: 项目根目录  
**产出**: `knowledge-base/.manifest.yaml` → `project_profile:` 区块 + 种子策略配置

##### 检测信号与判定逻辑

系统从四个维度收集信号，多信号投票决定项目类型：

| 维度 | 检测内容 | 示例 |
|------|---------|------|
| 包管理文件 | `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml` | 确定语言和运行时 |
| 框架依赖 | `dependencies` / `requirements` 中的框架包 | express, react, commander |
| 入口文件特征 | `app.listen()` vs `export {}` vs `commander` | Express server vs library vs CLI |
| 目录结构 | `src/routes/` vs `src/commands/` vs `src/components/` | 项目组织模式 |

**判定优先级**（第一个匹配 ≥2 个信号即停止）:

```
优先级 1: web-api     ← HTTP 框架依赖 (express, fastify, koa, gin, fastapi, django-rest)
优先级 2: cli         ← CLI 框架依赖 (commander, yargs, click, cobra, clap)
优先级 3: frontend    ← UI 框架依赖 (react, vue, svelte, angular, next.js)
优先级 4: library     ← 入口文件无 main loop，仅有 export 语句
优先级 5: data-pipeline ← ETL/workflow 框架依赖
优先级 6: generic     ← fallback — 无法匹配以上任何类型
```

##### 种子策略选择

| 项目类型 | 种子来源 | 提取方式 |
|---------|---------|---------|
| **web-api** | HTTP 路由注册 | `router.get/post/put/delete()` + `app.use(prefix, ...)` |
| **cli** | CLI 命令/子命令定义 | `.command()`, `@click.command()`, `cobra.Command` |
| **frontend** | 页面路由 + 顶层组件 | React Router `<Route>`, Vue Router `createRouter`, Next.js `pages/` |
| **library** | 公开导出 API（index 桶文件） | `export { ... } from` / `__all__` / `pub mod` |
| **data-pipeline** | 数据流入口函数 | ETL pipeline 定义 / DAG 节点 |
| **generic** | 降级为改进版目录扫描 | 仍按目录分组，但额外使用调用图和命名信号子聚类 |

##### 分类体系选择

不同项目类型使用不同的业务功能分类标签：

| 项目类型 | 分类体系 |
|---------|---------|
| **web-api** | core / admin / infrastructure / foundation |
| **cli** | command / utility / infrastructure / foundation |
| **frontend** | page / component / infrastructure / foundation |
| **library** | public-api / internal / utility / foundation |
| **data-pipeline** | source / transform / sink / infrastructure / foundation |
| **generic** | business / infrastructure / foundation |

##### 技术栈识别

```
输出示例:
  language: typescript
  runtime: node
  framework: express
  database: sqlite (better-sqlite3)
  auth_library: jsonwebtoken, bcryptjs
  validation_library: zod
```

此信息写入 manifest 的 `project_profile:` 字段，供后续阶段参考。

##### 产出：project_profile 区块

```yaml
# knowledge-base/.manifest.yaml（部分）
project_profile:
  type: web-api
  language: typescript
  runtime: node
  framework: express
  database: sqlite
  seed_strategy: api-route
  classification: web-api
  detection_signals:
    - dimension: framework
      signal: "express package detected"
    - dimension: entry_point
      signal: "app.listen() found in app.ts"
```

**门禁 G1.0** — 项目感知有效性 🆕:
- 规则: `project_profile.type` 非空
- 规则: `seed_strategy` 与检测到的类型匹配
- 规则: 检测信号 ≥ 2 个维度
- 失败动作: 降级为 `generic` + 记录 WARN

#### 1.2.2 代码扫描（多信号业务功能聚类）🔄

**输入**: 项目根目录 + 1.2.1 的种子策略和分类体系  
**产出**: `knowledge-base/.manifest.yaml` → `business_functions:` 区块 + `shared_foundations:` 区块

> **原则**: 不使用物理目录作为模块边界。通过七阶段聚类算法从代码中自动识别业务功能。

##### Phase 1: 种子收集（策略自适应）

根据 1.2.1 选择的种子策略从对应来源提取入口点。输出格式统一为 `{seed_id, entry_point, handler_symbol, imported_symbols}`。

**web-api 示例**:
```yaml
# 中间产物 — 种子清单
seeds:
  - id: seed-1
    entry: "POST /api/auth/login"
    handler: "auth/routes.ts:loginHandler"
    imports: [login, record, loginSchema]
    mount: "app.ts:app.use('/api/auth', authRoutes)"
  - id: seed-2
    entry: "GET /api/admin/login-logs"
    handler: "logging/routes.ts:adminQueryHandler"
    imports: [query, writeAuditLog, authenticate, requireAdmin]
```

**其他类型**: cli 提取 `{command, handler, imports}`，frontend 提取 `{path, component, imports}`，library 提取 `{symbol, kind, imports}`，generic 降级为所有导出函数。

##### Phase 2: 实体锚点识别（语言自适应）

从类型定义和数据模型文件中提取实体作为聚类锚点。通过 `codegraph_callers` 查找操作同一实体的文件。

- **TypeScript**: `interface`, `type`, SQL `CREATE TABLE`
- **Python**: `dataclass`, `NamedTuple`, SQLAlchemy model
- **Go**: `struct`, `type X interface`
- **Rust**: `struct`, `enum`, `trait`

```yaml
# 中间产物 — 实体锚点
entity_anchors:
  - name: LoginLog
    defined_in: src/types.ts
    operated_on_by: [db/models.ts, logging/service.ts, logging/routes.ts, auth/routes.ts]
    db_table: login_logs
  - name: User
    defined_in: src/types.ts
    operated_on_by: [auth/service.ts, auth/middleware.ts, auth/routes.ts]
```

##### Phase 3: 调用图邻近扩展（语言无关）

对每个种子沿调用图向外扩展 2-3 层深度。使用 `codegraph_callers` 递归追踪传递依赖。Phase 3 权重最高（仅次于种子），因为调用关系是最强的功能归属信号。

##### Phase 4: 导入邻近矩阵（机制通用）

构建 N×N 共现矩阵，Jaccard 相似度度量文件间的导入关系。阈值 > 0.5 的文件归为一组。仅导入语法因语言而异（import/require/use/include），机制不变。

##### Phase 5: 命名约定强化（机制通用）

按语义前缀分组符号。仅命名风格因语言而异（camelCase/snake_case/PascalCase），分组逻辑通用。

```yaml
# 中间产物 — 命名组
naming_prefixes:
  - prefix: "login"    → symbols: [login, loginLimiter, loginSchema, loginHandler]
  - prefix: "token"    → symbols: [createToken, verifyToken]
  - prefix: "audit"    → symbols: [writeAuditLog, AuditLog]
  - prefix: "cleanup"  → symbols: [startCleanupJob, runCleanup, deleteOldLoginLogs]
```

##### Phase 6: 加权投票合并与去重

| 信号来源 | 权重 | 说明 |
|---------|------|------|
| Phase 1 种子 | 3.0 | 最强的归属信号——路由入口定义了功能边界 |
| Phase 3 调用图 | 2.0 | 调用关系是仅次于种子的强信号 |
| Phase 4 导入邻近 | 1.5 | 导入模式反映模块耦合 |
| Phase 5 命名约定 | 1.0 | 辅助信号——命名不一定反映真实归属 |

**阈值**: 加权总分 ≥ 4.0 分配到业务功能。共享文件（归属 >1 功能，如 `logging/service.ts` 同时属于 login-audit-logging 和 data-cleanup）显式标注角色。

##### Phase 7: 分类（体系自适应）

根据 1.2.1 选择的分类体系为每个业务功能打标签。分类体系因项目类型而异，但聚类结果本身不受影响。

```yaml
# 产出: business_functions: 区块
business_functions:
  - name: "user-authentication"
    category: core
    confidence: 0.92
    seeds: [{type: route, source: "auth/routes.ts:POST /login"}]
    core_symbols: {functions: [login, createToken, verifyToken], types: [User]}
    primary_files: [src/auth/routes.ts, src/auth/service.ts, src/auth/middleware.ts]
    shared_files:
      - {file: src/logging/service.ts, role: "records_login_events", also_in: [login-audit-logging]}
    clustering_signals: {seed: true, call_graph: true, import_proximity: true, naming: true}
    size: {primary_files: 3, shared_files: 5, total_files: 6, kb_tokens_estimate: 12000}
```

**门禁 G1.1** — 业务功能覆盖率（更新规则）:
- 规则: `files_assigned / total_src_files >= 0.90`
- 规则: `seeds_attributed / total_seeds >= 0.95`（web-api=端点覆盖率, cli=命令覆盖率, generic=文件覆盖率）
- 规则: `orphan_files <= 3`
- 失败动作: 降低 Phase 6 阈值重聚类，或手动指定种子

#### 1.2.3 模块深度分析（并行）

**输入**: 业务功能清单  
**产出**: 每个业务功能的 KB 条目

> **细粒度拆分策略见第 14.2 节**: 单业务功能上下文控制在 < 50K tokens，超大功能进一步拆分为子功能。

对每个业务功能并行调用，每个 agent 内部使用 `codegraph_node` (MCP) 获取符号信息：
```
# 通过 dispatching-parallel-agents 分派，每个 agent 处理一个业务功能
# agent 内部: codegraph_node 分析该功能的所有 primary_files + shared_files
```

每个业务功能产出:
```
knowledge-base/business-functions/<bf-name>/
├── overview.md       # 业务功能概述、职责、聚类证据
├── api.md            # 对外接口（路由端点/命令/公开API）
├── data-model.md     # 数据模型/类型定义（含共享声明）
├── business-logic.md # 核心业务逻辑描述
└── dependencies.md   # 跨业务功能依赖关系
```

**门禁 G1.2** — KB 条目完整性（更新规则）:
- 规则: 每个业务功能目录下至少存在 `overview.md` + `api.md`
- 规则: 若旧 `knowledge-base/modules/` 存在，标记为 `stale`
- 失败动作: 标记缺失业务功能，重新分析

#### 1.2.4 KB 汇总整合

**输入**: 各业务功能 KB 条目  
**产出**: 项目级 KB 汇总 + 业务功能依赖图

```
knowledge-base/
├── architecture/
│   ├── overview.md        # 整体架构描述
│   ├── components.md      # 业务功能拓扑
│   └── data-flow.md       # 数据流描述
├── apis/
│   ├── internal.md        # 内部 API 总览
│   └── external.md        # 外部 API 总览
└── data-models/
    └── global.md          # 全局数据模型
```

**业务功能依赖图**: 从 manifest 的 `depends_on` 字段自动生成，展示业务功能间的调用/数据/中间件依赖链。

**门禁 G1.3** — KB 一致性:
- 规则: 汇总文档中的引用与实际业务功能 KB 条目一致
- 实现: 解析所有 `[[bf-name]]` 引用，验证目标存在

#### 1.2.5 需求反向推断

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
module: auth-service              # v1 旧模块名，向后兼容
business_function: user-authentication  # 🆕 v2 业务功能名
status: baseline
inferred_from:
  - src/auth/service.ts
  - src/auth/routes.ts
  - src/auth/middleware.ts
confidence: high
source: auto-inferred
usage_restriction: reference_only  # reference_only = 仅供参考，禁止作为新功能设计的强制约束
created: 2026-06-27
---
> ⚠️ **AUTO-INFERRED BASELINE** | confidence: high | source: code reverse-engineering
>
> This requirement was inferred from existing code implementation, NOT from original design documents or specifications. It may not reflect original design intent.
>
> **Usage restriction (`reference_only`)**: This baseline requirement serves as **documentation reference only**. It MUST NOT be used as a mandatory constraint when designing new features in Stage 3. If a new requirement conflicts with a baseline requirement, the baseline requirement gives way.

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

**基线需求用途限制规则**:
- 基线需求 frontmatter 必须包含 `source: auto-inferred`、`usage_restriction: reference_only` 和 `business_function:`（v2 新增）
- `business_function:` 字段关联 v2 业务功能名，同时保留 `module:` 字段指向 v1 旧模块名以实现向后兼容
- 门禁 G2.4 检查 `source` 和 `usage_restriction` 字段存在性
- 门禁 G3.2 检查设计方案未将 `usage_restriction: reference_only` 的需求作为强制约束

**门禁 G1.4** — 需求覆盖率（更新规则）:
- 规则: 每个 core/admin/command/page/public-api 类别的业务功能至少被 1 个需求覆盖
- 规则: infrastructure 和 foundation 类别的业务功能免于需求覆盖要求（它们是支持性基础设施）
- 额外规则: `confidence: low` 的需求不超过 20%

#### 1.2.6 门禁汇总校验

**门禁 G1.5** — 阶段一总门禁（所有子门禁通过 + 以下规则）:
- [ ] G1.0: 项目感知有效性通过（project_profile.type 非空，信号 ≥ 2 维度）
- [ ] G1.1: 业务功能覆盖率 >= 90%（文件覆盖率） + 入口覆盖率 >= 95%
- [ ] G1.2: 所有业务功能 KB 条目完整
- [ ] G1.3: KB 一致性验证通过
- [ ] G1.4: 需求覆盖率达标（core/admin 类功能全覆盖）
- [ ] clustering_quality >= 0.80（加权平均置信度）
- [ ] orphan_files <= 3
- [ ] 所有产物已 `git add` 且无 merge conflict
- [ ] KB 总大小 > 0（防止空提交）

#### 1.2.7 Git 提交

```bash
git checkout -b kb/update-$(date +%Y-%m-%d)
git add knowledge-base/ requirements/baseline/
git commit -m "[devloop] stage:1 action:reverse complete:$(date)"
```

---

## 2. 阶段二：需求摄入

### 2.1 总编排 Skill: `devloop-intake`

```
devloop-intake (🆕 编排 Skill，~180 行)
├── 2.1 输入解析        → (编排层逻辑) 🆕
├── 2.2 KB 上下文加载   → codegraph_explore ✅ 已有 (MCP) + (索引逻辑) 🆕
├── 2.3 影响分析        → brainstorming ✅ 已有
├── 2.4 非功能需求评估  → nfr-checklist 模板 🆕 （强制步骤，不可跳过）
├── 2.5 需求文档生成    → openspec-propose ✅ 已有
├── 2.6 门禁校验        → devloop-guard 🆕 (T0.3)
├── 2.7 Git 提交        → git (Bash) ✅ 已有
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

#### 2.2.4 非功能需求强制评估 🆕

> **设计动机**: 安全/性能/可访问性/合规等非功能需求无法从代码逆向推断（代码记录实现而非约束意图）。KB 对这些维度天然沉默，下游 LLM 会将沉默解读为「不需要」。此步骤在所有需求摄入时强制执行，不依赖 KB 内容。

**输入**: 影响分析报告 + 需求类型/优先级 + `templates/nfr-checklist.md`
**产出**: 影响分析报告的「非功能需求评估」章节

**评估维度**（`templates/nfr-checklist.md` 定义的强制检查清单）:

| 维度 | 触发条件 | 必须产出的内容 |
|------|---------|--------------|
| **认证/授权 (AuthN/Z)** | `type ∈ [feature, enhancement]` 且涉及用户操作 | 需要的权限级别（匿名/登录/角色）、新增权限点 |
| **输入校验 (Validation)** | 涉及 API 变更或数据模型变更 | 每个新字段的类型/范围/长度约束、SQL 注入/XSS 防护要求 |
| **数据保护 (Data Privacy)** | 涉及用户数据或 `type ∉ [chore]` | 敏感字段（PII/密码/Token）的存储/传输/日志脱敏策略 |
| **速率限制 (Rate Limiting)** | 涉及新 API 端点 | 是否需要限流、限流粒度（用户级/IP级） |
| **审计日志 (Audit)** | `priority ∈ [high, medium]` 或涉及资金/权限操作 | 需要记录的操作类型、日志中禁止包含的字段 |
| **性能 (Performance)** | `type = feature` 且涉及数据查询 | 预期数据量级、是否需要分页/缓存/异步 |
| **可访问性 (Accessibility)** | 涉及前端 UI 变更 | WCAG 级别要求、键盘导航/屏幕阅读器支持 |

**评估流程**:
```
1. 加载 templates/nfr-checklist.md
2. 根据需求的 type/priority/涉及模块，判定每个维度的触发条件
3. 对触发的每个维度:
   a. 分析当前需求描述中是否已覆盖该约束
   b. 如未覆盖 → 生成具体的补充验收标准
   c. 如已覆盖 → 标记 ✅ 并引用原文位置
4. 汇总为「非功能需求评估」章节，追加到影响分析报告末尾
5. 将每个触发的维度对应的验收标准注入需求文档的「验收标准」部分
```

**评估产出示例**:
```markdown
## 非功能需求评估

> 根据 `templates/nfr-checklist.md` 强制执行，评估时间: 2026-06-27T10:30:00Z

### 认证/授权
- 触发: ✅ (feature 类型 + 涉及用户操作)
- 补充验收标准:
  - [ ] OAuth2 回调接口不需要登录态（匿名访问），但需验证 state 参数防 CSRF
  - [ ] 已有登录接口需增加 OAuth2 选项，保持现有认证逻辑不变

### 输入校验
- 触发: ✅ (涉及 API 变更)
- 补充验收标准:
  - [ ] OAuth2 authorization code 长度限制 ≤ 1024 字符
  - [ ] redirect_uri 必须白名单校验，防止开放重定向

### 数据保护
- 触发: ✅ (涉及用户数据)
- 补充验收标准:
  - [ ] oauth_token 和 oauth_uid 存储时加密（AES-256-GCM）
  - [ ] 日志中不得输出完整的 oauth_token（仅记录前8字符哈希）

### 速率限制
- 触发: ✅ (新增 API 端点)
- 补充验收标准:
  - [ ] POST /api/auth/oauth2/callback 每 IP 每分钟 ≤ 10 次

### 审计日志
- 触发: ✅ (priority=high + 涉及认证操作)
- 补充验收标准:
  - [ ] 记录每次 OAuth2 登录事件（用户ID、provider、时间戳、IP）
  - [ ] 日志中禁止包含 oauth_token 和 oauth_uid 明文
```

**门禁 G2.3-NFR** — NFR 评估完整性（追加到 G2.3 hard checks）:
- 规则: 影响分析报告必须包含「非功能需求评估」章节
- 规则: 所有触发条件满足的维度，都必须在需求文档的「验收标准」中有对应的 checklist 项
- 规则: 未触发的维度必须注明「不适用」及原因（一句即可）
- 失败动作: block_and_retry（触发补充评估）

**不适用声明**: 如果某个维度确实不适用，标注原因即可通过——不要求无意义的填表。例如 `type: chore` 且无 API 变更，可声明「NFR-输入校验: 不适用（chore，无 API 变更）」。

#### 2.2.5 需求文档生成

调用 `openspec-propose` 生成正式需求（原 2.2.4，编号因插入 NFR 步骤而后移）：

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
- 规则: frontmatter 包含所有必填字段（含 `source` 和 `usage_restriction`）
- 规则: 需求文档与 OpenSpec proposal.md 内容一致
- 规则: `kb_context` 引用有效且不少于 2 个
- 规则: 验收标准中包含了所有 NFR 评估触发的补充验收项

#### 2.2.6 门禁汇总校验

**门禁 G2.5** — 阶段二总门禁:
- [ ] G2.1: 输入有效性通过
- [ ] G2.2: KB 上下文相关性通过（或标记 low confidence）
- [ ] G2.3: 影响分析完整（hard checks 全部通过）
- [ ] G2.3-NFR: 非功能需求评估完整（所有触发维度已产出验收标准）
- [ ] G2.4: 需求文档结构正确（含 NFR 补充验收项）
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

**门禁分类原则**：

| 类型 | 判定方式 | 阻断权限 | 说明 |
|------|---------|---------|------|
| **硬门禁 (hard)** | 脚本自动判定（`test -f`、`grep -c`、退出码等） | 可 BLOCK | 事实性检查，不存在歧义 |
| **软门禁 (soft)** | LLM 评估语义质量 | 仅 WARN | 语义判断有不确定性，不能作为阻断依据 |

```yaml
# 门禁配置 — 可项目级覆盖
gates:
  stage-1-reverse:
    - id: G1.0
      name: 项目感知有效性 🆕
      type: hard                                           # 硬门禁：字段非空 + 信号计数
      checks:
        - "project_profile.type 非空"
        - "seed_strategy 与 project_profile.type 匹配"
        - "detection_signals 维度 >= 2"
      on_fail: block_and_retry                             # 失败降级为 generic + WARN
    - id: G1.1
      name: 业务功能覆盖率
      type: hard                                           # 硬门禁：文件计数 + 入口覆盖率
      checks:
        - "files_assigned_to_at_least_one_bf / total_src_files >= 0.90"
        - "seeds_attributed / total_seeds >= 0.95"         # web-api=端点, cli=命令, generic=文件
        - "orphan_files <= 3"
      on_fail: warn_and_continue
    - id: G1.2
      name: KB 条目完整性
      type: hard                                           # 硬门禁：检查文件存在性
      checks:
        - "all(bf.has('overview.md') for bf in business_functions)"
        - "all(bf.has('api.md') for bf in business_functions)"
        - "if legacy modules/ exists: mark as stale"
      on_fail: block_and_retry
    - id: G1.3
      name: KB 内部一致性
      type: mixed
      hard:                                                # 硬：检查交叉引用目标文件存在
        - "all_wiki_link_targets_exist(knowledge-base/)"
      soft:                                                # 软：引用内容语义是否匹配（LLM评估）
        - "交叉引用指向的业务功能描述与引用上下文一致"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue
    - id: G1.4
      name: 需求反向覆盖率
      type: hard                                           # 硬门禁：计数（排除 infrastructure/foundation）
      checks:
        - "business_functions_covered_by_req(core+admin+command+page+public-api) / total_target_bfs >= 0.90"
        - "infrastructure 和 foundation 类别免于需求覆盖"
      on_fail: warn_and_continue
    - id: G1.5
      name: 阶段一总门禁
      type: hard                                           # 汇总硬门禁 + 聚类质量 + 孤儿文件
      checks:
        - "G1.0 = pass AND G1.2 = pass AND G1.3.hard = pass"
        - "weighted_average_confidence_across_bfs >= 0.80"
        - "orphan_files <= 3"

  stage-2-intake:
    - id: G2.1
      name: 输入有效性
      type: hard                                    # 硬：检查字段非空 + 枚举值
      checks:
        - "input.title 非空"
        - "input.description 非空"
        - "input.type ∈ [feature, bugfix, enhancement, refactor, chore]"
      on_fail: reject

    - id: G2.2
      name: KB 上下文相关性
      type: soft                                    # 软：语义匹配质量 LLM 判定
      checks:
        - "匹配到的 KB 模块至少 1 个"
        - "匹配模块的关键词与需求描述语义相关"
      on_fail: warn_and_continue                    # 软门禁只警告，新模块可能没有 KB

    - id: G2.3
      name: 影响分析完整性
      type: mixed
      hard:                                         # 硬：必须包含的章节存在
        - "影响分析文档存在且非空"
        - "包含 '涉及模块' 章节"
        - "包含 'API变更' 或明确声明无变更"
        - "包含 '数据模型变更' 或明确声明无变更"
        - "包含 '非功能需求评估' 章节（由 NFR checklist 强制产出）"
        - "所有 NFR 触发维度在需求文档验收标准中有对应 checklist 项"
      soft:                                         # 软：章节内容质量（LLM评估）
        - "涉及模块的分析深度足够（非一句话敷衍）"
        - "API/数据模型变更描述具体到函数/字段级别"
        - "NFR 评估各维度的补充验收标准具体可测（非泛泛而谈）"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G2.4
      name: 需求文档结构
      type: hard                                    # 硬：frontmatter 必填字段
      checks:
        - "frontmatter 包含 id, title, type, priority, status, module"
        - "kb_context 引用至少 2 个（或声明理由为 new_module）"
        - "需求文档内容与 openspec proposal.md 一致"
      on_fail: block_and_retry

    - id: G2.5
      name: 阶段二总门禁
      type: hard
      checks:
        - "G2.1 = pass AND G2.3.hard = pass AND G2.3-NFR = pass AND G2.4 = pass"

  stage-3-build:
    - id: G3.1
      name: 需求可构建性
      type: hard                                    # 硬：状态字段 + 文件存在
      checks:
        - "需求 status ∈ [proposed, designed, planned]"
        - "需求 kb_context 中引用的所有文件仍然存在"
      on_fail: block_and_retry                      # 不满足则跳过该需求

    - id: G3.2
      name: Design 阶段质量
      type: mixed
      hard:                                         # 硬：产物存在性
        - "design.md 存在且非空"
        - "delta spec 文件存在"
        - "comet-guard 返回 ALL CHECKS PASSED"
        - "设计方案未将任何 baseline 需求（usage_restriction: reference_only）作为强制约束"
        - "如果设计方案与某个 baseline 需求冲突，已明确记录偏离理由"
      soft:                                         # 软：设计质量（LLM评估）
        - "设计方案不违反 architecture/overview.md 中的架构约束"
        - "delta spec 覆盖了影响分析中的所有涉及模块"
        - "API 契约变更与 KB 中记录的现有契约兼容或明确标注 breaking change"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G3.3
      name: Plan 阶段质量
      type: mixed
      hard:
        - "所有 task 有唯一编号"
        - "每个 task 关联到具体的 spec delta"
        - "无循环依赖（task 引用链无环）"
      soft:
        - "task 粒度合理（预估每个 task < 200 行变更）"
        - "task 顺序符合依赖逻辑"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G3.4
      name: Build 阶段质量
      type: hard                                    # 硬：task 完成状态 + 测试结果
      checks:
        - "所有 task checkbox 已勾选"
        - "测试覆盖率不低于变更前"
        - "无 linter 错误"
        - "devloop-guard build ALL CHECKS PASSED"
      on_fail: block_and_retry

  stage-4-verify:
    - id: G4.1
      name: 自动化验证
      type: hard
      checks:
        - "所有测试通过（退出码=0）"
        - "构建成功（退出码=0）"
        - "Lint / Type Check 零错误（退出码=0）"
      on_fail: block_and_retry

    - id: G4.2
      name: 代码审查
      type: mixed
      hard:
        - "审查报告存在"
        - "无 Critical 级别问题（通过审查输出计数判定）"
      soft:                                         # 软：审查内容的充分性
        - "审查覆盖了正确性/安全性/性能/可维护性/复用性五个维度"
        - "审查意见有具体代码位置引用"
      on_fail:
        hard: block_and_retry
        soft: warn_and_continue

    - id: G4.3
      name: 修复循环限制
      type: hard
      checks:
        - "修复循环次数 < 3"
        - "每次修复后不引入新的 Critical 问题"
      on_fail: pause_for_human

    - id: G4.4
      name: 阶段四总门禁
      type: hard
      checks:
        - "G4.1 = pass AND G4.2.hard = pass AND G4.3 未超限"
        - "需求文档中的所有验收标准全部满足"
        - "KB 一致性：代码变更未破坏 KB 中的 API 契约"
        - "Git 状态干净（无未暂存变更）"

  stage-5-loop:
    - id: G5.1
      name: 闭环状态一致性
      type: hard
      checks:
        - "需求文档 frontmatter.status 与所在目录一致"
        - "loop-state.yaml 中的 active_requirement 指向有效需求"
        - "无孤儿需求（文档存在但未被任何索引引用）"

# on_fail 动作说明:
#   block_and_retry: 阻断流程，自动重试（最多3次）
#   warn_and_continue: 记录警告，继续流程
#   reject: 直接拒绝，返回给调用方
#   pause_for_human: 暂停等待人工决策
#
# 软门禁规则:
#   - soft 类型的门禁只能使用 warn_and_continue 或 pause_for_human
#   - 使用 warn_and_continue 则记录到 .drift-report.yaml 的 soft_gate_warnings 字段
#   - 软门禁警告在阶段总门禁汇总时展示，但不阻断流程
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

**两阶段执行**: 硬门禁先跑（脚本自动），全部通过后软门禁交给 LLM 评估。

```python
# 伪代码: 门禁执行引擎（硬/软分离）
def execute_gate(gate_config, context):
    result = GateResult()

    # === 第一阶段：硬门禁（脚本自动判定）===
    if gate_config.type in ("hard", "mixed"):
        hard_checks = gate_config.hard if gate_config.type == "mixed" else gate_config.checks
        for check in hard_checks:
            check_result = run_hard_check(check, context)
            if not check_result.passed:
                action = gate_config.on_fail if gate_config.type == "hard" else gate_config.on_fail["hard"]
                if action == "block_and_retry":
                    for attempt in range(3):
                        context = auto_fix(gate_config, check_result)
                        if run_hard_check(check, context).passed:
                            break
                    else:
                        result.hard_failures.append(check_result)
                        result.blocked = True
                elif action == "reject":
                    result.hard_failures.append(check_result)
                    result.blocked = True
                    return result  # reject 不重试
                elif action == "warn_and_continue":
                    result.warnings.append(check_result)
                elif action == "pause_for_human":
                    result.pending_human.append(check_result)

    if result.blocked:
        return result

    # === 第二阶段：软门禁（LLM 语义评估）===
    # 仅当硬门禁全部通过后才执行
    if gate_config.type in ("soft", "mixed"):
        soft_checks = gate_config.soft if gate_config.type == "mixed" else gate_config.checks
        for check in soft_checks:
            # 软门禁委托 LLM agent 评估，输出结构化结果
            llm_result = llm_evaluate(check, context, schema=SOFT_GATE_SCHEMA)
            if not llm_result.passed:
                # 软门禁只能 WARN，不能 BLOCK
                result.warnings.append(SoftGateWarning(
                    gate=gate_config.id,
                    check=check,
                    reason=llm_result.reason,
                    suggestion=llm_result.suggestion
                ))

    return result

# 软门禁结构化输出 schema
SOFT_GATE_SCHEMA = {
    "passed": "boolean — 检查是否通过",
    "reason": "string — 不通过的具体原因，引用具体文件位置",
    "suggestion": "string — 修复建议"
}
```

**软门禁核心原则**:
- 软门禁 NEVER 使用 `block_and_retry`，只能 `warn_and_continue` 或 `pause_for_human`
- 软门禁警告写入 `knowledge-base/.drift-report.yaml` 的 `soft_gate_warnings` 字段
- 阶段总门禁汇总时展示所有软门禁警告，但即使有警告也不阻断，仅提示用户关注

### 7.3 完整门禁矩阵

| 阶段 | 门禁ID | 名称 | 类型 | 判定方式 | 阻断级别 | 自动修复 |
|------|--------|------|------|---------|---------|---------|
| 1 | G1.0 | 项目感知有效性 🆕 | hard | 脚本（字段非空+信号计数） | BLOCK | Y(降级generic) |
| 1 | G1.1 | 业务功能覆盖率 | hard | 脚本（文件计数+入口覆盖率） | WARN | N |
| 1 | G1.2 | KB条目完整性 | hard | 脚本（文件存在） | BLOCK | Y |
| 1 | G1.3 | KB内部一致性 | mixed | hard:脚本(引用目标存在) / soft:LLM(语义匹配) | hard=BLOCK, soft=WARN | Y(hard) |
| 1 | G1.4 | 需求反向覆盖率 | hard | 脚本（计数，排除infrastructure/foundation） | WARN | N |
| 1 | G1.5 | 阶段一总门禁 | hard | 汇总硬门禁+clustering_quality+orphan_files | BLOCK | N |
| 2 | G2.1 | 输入有效性 | hard | 脚本（字段/枚举） | REJECT | N |
| 2 | G2.2 | KB上下文相关性 | soft | LLM（语义匹配） | WARN | N |
| 2 | G2.3 | 影响分析完整性 | mixed | hard:脚本(章节存在) / soft:LLM(内容质量+安全维度) | hard=BLOCK, soft=WARN | Y(hard) |
| 2 | G2.4 | 需求文档结构 | hard | 脚本（frontmatter字段） | BLOCK | Y |
| 2 | G2.5 | 阶段二总门禁 | hard | 汇总硬门禁结果 | BLOCK | N |
| 3 | G3.1 | 需求可构建性 | hard | 脚本（状态+文件存在） | BLOCK | N |
| 3 | G3.2 | Design阶段质量 | mixed | hard:脚本(文件存在+guard) / soft:LLM(架构约束+契约兼容) | hard=BLOCK, soft=WARN | N |
| 3 | G3.3 | Plan阶段质量 | mixed | hard:脚本(编号+依赖) / soft:LLM(粒度+顺序) | hard=BLOCK, soft=WARN | Y(hard) |
| 3 | G3.4 | Build阶段质量 | hard | 脚本（task完成+测试+linter） | BLOCK | Y |
| 4 | G4.1 | 自动化验证 | hard | 脚本（退出码） | BLOCK | Y |
| 4 | G4.2 | 代码审查 | mixed | hard:脚本(报告存在+Critical计数) / soft:LLM(审查充分性) | hard=BLOCK, soft=WARN | Y(hard) |
| 4 | G4.3 | 修复循环限制 | hard | 脚本（计数） | PAUSE | N |
| 4 | G4.4 | 阶段四总门禁 | hard | 汇总硬门禁结果 | BLOCK | N |
| 5 | G5.1 | 闭环状态一致性 | hard | 脚本（状态字段） | BLOCK | Y |

> **类型说明**: `hard` = 脚本自动判定，事实性检查，可阻断；`soft` = LLM 语义评估，仅 WARN 不阻断；`mixed` = 包含 hard + soft 两层，分别使用各自阻断规则。

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
| T4 | **非功能需求**: 性能/安全/可访问性需求难以从代码逆向 | 阶段一遗漏非功能需求 | ✅ 已解决 | 阶段二新增「非功能需求强制评估」步骤（2.2.4），加载 `templates/nfr-checklist.md` 遍历 7 个维度，触发条件驱动的补充验收标准生成 |
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

## 10bis. Manifest 迁移策略（v1 → v2）

### 10bis.1 迁移触发

当新版阶段一（v2 聚类）首次在已有目录式 manifest（`schema_version: "1.0"`）的项目上运行时自动触发。

### 10bis.2 迁移流程

1. **检测旧 manifest**: 检查 `knowledge-base/.manifest.yaml` 的 `schema_version` 字段。
   - `"1.0"` 或无 `business_functions:` 根键 → 触发迁移
   - `"2.0"` → 跳过迁移，执行增量更新

2. **归档旧数据**: 
   ```
   knowledge-base/modules/ → knowledge-base/.migrated-from-v1/modules/
   knowledge-base/.manifest.yaml → knowledge-base/.migrated-from-v1/manifest.yaml
   ```

3. **创建迁移记录**:
   ```yaml
   # knowledge-base/.migrated-from-v1/migration-record.yaml
   migration_date: "<ISO 8601>"
   previous_schema: "1.0"
   current_schema: "2.0"
   old_modules_mapping:
     src/auth/ → user-authentication, jwt-token-management
     src/db/ → database-management, login-audit-logging, data-cleanup
     src/logging/ → login-audit-logging, self-service-log-query, audit-trail, data-cleanup
     src/middleware/ → rate-limiting, error-handling, request-tracing
     src/app.ts → health-checking, data-cleanup, error-handling
     src/config.ts → shared-foundations
     src/types.ts → shared-foundations
   ```

4. **执行 v2 聚类**: 按七阶段聚类在 `src/` 上运行，生成 `business_functions:` 区块。

5. **更新基线需求**: 对 `requirements/baseline/` 中的每个需求：
   - 如果 `module:` 指向已拆分的旧模块 → 更新为新的 `business_function:` 名
   - 如果旧模块是 DevLoop 框架组件（comet-scripts 等）→ 标记为 `module: devloop-framework`，保持旧 `modules/` 结构

6. **生成迁移报告**: `knowledge-base/.migration-report.yaml`。

### 10bis.3 向后兼容行为

- 基线需求保留 `module:` 字段指向旧模块名（新增 `business_function:` 字段指向新功能名）
- `devloop-guard` 脚本检查 `business_functions:` 优先，fallback `modules:`
- 阶段二 KB 上下文加载基于文件路径匹配（`kb_context:` 引用实际 KB 文件路径，非模块名）
- DevLoop 框架组件（`.comet/scripts/`, `.claude/skills/`, `templates/`, `design/`）排除在业务功能聚类之外，保持旧 `modules/` 结构中

### 10bis.4 项目类型感知的扫描排除

`scan_exclude` 配置（可在 `.comet/guard-config.yaml` 覆盖）:
```yaml
scan_exclude:
  - ".comet/"
  - ".claude/"
  - "templates/"
  - "design/"
  - "node_modules/"
  - ".git/"
  - "knowledge-base/"       # 避免自指
  - "requirements/"
  - "dist/"
  - "build/"
  - "__pycache__/"
  - "*.test.*"
  - "*.spec.*"
```

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

### 14.2 逆向分析方向：项目类型感知的细粒度业务功能拆分

**原则**: 阶段一不分析"整个项目"，而是分析"每个最小可独立理解的业务功能"。

```
拆分策略 (v2):
  1. 1.2.1 项目感知 → 确定项目类型 + 种子策略 + 分类体系
  2. 1.2.2 多信号聚类 → 自动识别业务功能边界
  3. 每个业务功能独立分析，单功能上下文 < 50K tokens
  4. 超大业务功能（>30 文件或 token > 50K）进一步拆分为子功能
  5. 每个业务功能生成独立的 KB 条目（相互引用而非重复内容）

拆分粒度 (v2):
  - 按聚类结果拆分（信号驱动，非目录驱动）
  - 共享文件（归属 >1 业务功能）在多个 KB 条目中引用，不重复分析
  - infrastructure 和 foundation 类别不强制生成完整 5 文件 KB 条目
```

**并行分析**: 各业务功能完全独立，使用并行 agent 同时分析（阶段一已设计并行），单个 agent 只看到自己负责的业务功能代码（含 primary_files + shared_files）。

### 14.3 需求摄入方向：索引式按需加载

**原则**: 阶段二/三不加载完整 KB，只加载与当前需求相关的 KB 条目。

```
索引式加载流程:
  1. 需求输入 → 关键词提取（模块名、函数名、数据实体、API 端点）
  2. 关键词 → KB 索引查询 (.manifest.yaml business_functions 区块)
  3. 匹配结果排序（按相关度）
  4. 加载 Top-N 相关条目（N 由上下文预算决定）
  5. 如果匹配条目总 token 数 > 预算 → 加载摘要层（overview.md 优先于完整 api.md）

索引结构 (knowledge-base/.manifest.yaml v2):
  business_functions:
    - name: user-authentication
      api_endpoints: [POST /api/auth/login]
      core_symbols:
        functions: [login, createToken, verifyToken]
        types: [User]
      data_entities: [User]
      keywords: [auth, login, token, oauth, session, password]    # 保留向后兼容
      size:
        total_files: 6
        kb_tokens_estimate: 12000
        summary_tokens_estimate: 1500

  可检索字段（阶段二关键词匹配）:
    - business_function.name → "user-authentication"
    - api_endpoints[].path → "/api/auth/login"
    - core_symbols.functions[] → "login", "createToken"
    - core_symbols.types[] → "User"
    - data_entities[] → "User"
    - keywords[] → "auth", "login" (向后兼容)

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
