---
name: devloop-reverse
description: "DevLoop 阶段一：逆向分析 (v2) — 项目感知→多信号业务功能聚类→并行分析→KB汇总→需求反推→门禁→提交。自动识别业务功能，不依赖物理目录布局。"
---

# DevLoop Reverse — 阶段一：逆向分析 (v2)

从现有代码库生成结构化的知识库（KB）和基线需求（baseline requirements）。
v2 使用**项目类型感知 + 多信号业务功能聚类**替代物理目录式模块划分，
适用于 web-api / cli / frontend / library / data-pipeline / generic 等任意项目类型。

所有产物自动标记 `source: auto-generated`，基线需求标记 `usage_restriction: reference_only`。

## 触发条件

1. 新项目启动时，首次构建知识库
2. 收到 `/devloop reverse` 命令（全量或 `--scope <bf>` 增量）
3. KB 漂移检测触发增量重建（`drift_score > 0.3`）
4. 手动触发保鲜：`/devloop kb-fresh --mode fix`

## 流程总览

```
Step 1: 项目感知         → codegraph_explore (MCP)         → project_profile 区块
Step 2: 多信号聚类       → codegraph_explore + codegraph_callers → business_functions 清单
Step 3: 并行业务功能分析 → dispatching-parallel-agents      → knowledge-base/business-functions/<name>/
       (每个 agent)      → codegraph_node (MCP)
Step 4: KB 汇总整合      → (编排层 LLM)                    → architecture/ + apis/ + data-models/
Step 5: 需求反向推断     → brainstorming                    → requirements/baseline/REQ-*.md
Step 6: 门禁校验         → devloop-guard check stage-1     → G1.0 - G1.5
Step 7: Git 提交         → git commit                       → [devloop] stage:1 action:reverse
```

每个 Step 结束后调用 `comet-state checkpoint` 保存断点，支持中断恢复。

## Step 1: 项目感知 (1.2.1)

**目标**: 检测项目类型和技术栈，选择对应的种子策略和分类体系。

### 1.1 检测信号采集

使用 `codegraph_explore` 从 4 个维度检测项目特征：

| 维度 | 检测目标 | 示例 |
|------|---------|------|
| 包管理器 | `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml` | express / fastify / commander / react |
| 框架依赖 | HTTP/CLI/UI 框架依赖 | express → web-api, commander → cli |
| 入口文件 | `app.listen()` vs `export` vs `.command()` | `src/app.ts:app.listen(3000)` |
| 目录结构 | `src/routes/` vs `src/commands/` vs `src/components/` | 路由目录 → web-api |

### 1.2 项目类型判定

按优先级匹配（第一个命中即停止）：

| 项目类型 | 判定信号 | 种子策略 |
|---------|---------|---------|
| **web-api** | HTTP 框架 (express/fastify/koa/gin/fastapi/django-rest) | `api-route` — 扫描路由注册 |
| **cli** | CLI 框架 (commander/yargs/click/cobra/clap) | `cli-command` — 扫描命令定义 |
| **frontend** | UI 框架 (react/vue/svelte/angular/next.js) | `page-route` — 扫描页面路由/顶层组件 |
| **library** | 入口文件仅 export，无 main loop | `public-api` — 扫描桶文件公开导出 |
| **data-pipeline** | ETL/workflow 框架依赖 | `dataflow-entry` — 扫描 pipeline/DAG 定义 |
| **generic** | 不匹配以上任何类型 | `directory-plus` — 退化版目录扫描 + 调用图子聚类 |

多信号投票：至少 2 个维度一致才确认类型；不确定时降级为 generic 并发出 warning。

### 1.3 技术栈识别

提取框架、语言、数据库、认证库等元信息，写入 manifest `project_profile:`。

### 1.4 分类体系选择

| 项目类型 | 业务功能分类体系 |
|---------|-----------------|
| web-api | core / admin / infrastructure / foundation |
| cli | command / utility / infrastructure / foundation |
| frontend | page / component / infrastructure / foundation |
| library | public-api / internal / utility / foundation |
| data-pipeline | source / transform / sink / infrastructure / foundation |
| generic | business / infrastructure / foundation |

**门禁 G1.0** — 项目感知有效性:
- `project_profile.type` 非空
- `seed_strategy` 与 `project_profile.type` 匹配
- `detection_signals` 维度 >= 2

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.1 "Project perception complete: type=<type>, seed_strategy=<strategy>"
```

## Step 2: 多信号业务功能聚类 (1.2.2)

**目标**: 自动识别业务功能边界，不依赖物理目录结构。

### 2.1 七阶段聚类算法

#### Phase 1: 种子收集（策略自适应）

根据 Step 1 选择的种子策略提取入口点：

- **api-route**: 扫描路由文件 → `{method, path, handler, imports}`，记录 mount 关系
- **cli-command**: 扫描命令定义 → `{command, subcommand, handler, imports}`
- **page-route**: 扫描路由/组件 → `{path, component, imports}`
- **public-api**: 扫描公开导出 → `{symbol, kind, imports}`
- **directory-plus**: 扫描所有导出函数作为种子

输出统一格式: `{seed_id, entry_point, handler_symbol, imported_symbols}`。

#### Phase 2: 实体锚点识别（语言自适应）

从类型定义和数据模型文件中提取核心实体：
- **TypeScript**: `interface`, `type`, SQL `CREATE TABLE`
- **Python**: `dataclass`, `NamedTuple`, SQLAlchemy `class X(Base)`
- **Go**: `struct`, `type X interface`
- **Rust**: `struct`, `enum`, `trait`

使用 `codegraph_callers` 查找操作同一实体的文件作为聚类锚点。

#### Phase 3: 调用图邻近扩展

对每个种子，沿调用图向外扩展（2-3 层深度），递归追踪传递依赖。
通过 `codegraph_callers` 实现，**语言无关**。

#### Phase 4: 导入邻近矩阵

构建文件共现矩阵，基于 Jaccard 相似度度量文件间导入关系。**机制通用**。

#### Phase 5: 命名约定强化

按语义前缀分组符号（如 `login*` / `auth*` / `user*`）。**机制通用**，命名风格因语言自适应。

#### Phase 6: 加权投票合并与去重

| 信号来源 | 权重 |
|---------|------|
| Phase 1 种子 | 3.0 |
| Phase 3 调用图 | 2.0 |
| Phase 4 导入邻近 | 1.5 |
| Phase 5 命名约定 | 1.0 |

阈值: ≥ 4.0 分配到业务功能。共享文件（归属 >1 个功能）显式标注角色和 `also_in` 列表。

#### Phase 6b: 缺口填补（基础设施/基础层兜底）

Phase 6 加权投票后，**无种子信号**的基础设施和基础层文件（middleware、config、db、error handler 等）可能得分低于 4.0 阈值。对未分配文件执行二次判定：

**前置规则 — 共享基础层自动检测**（在所有 gap-fill 之前执行）：

1. 被 ≥ 4 个不同 BF 导入的**纯类型/配置文件**（无函数导出）→ `shared_foundations`，confidence 0.99
2. 被 ≥ 3 个不同 BF 调用 **且** 命名信号指向 foundation BF（database/config）→ 分配到命名信号指向的 foundation BF，而非调用图占优的 BF
3. 被 ≥ 4 个不同 BF 调用的通用工具文件 → `shared_foundations`，confidence 0.90

| 剩余信号组合 | 动作 | confidence |
|------------|------|------------|
| **跨 BF 导入者 ≥ 4** 或 **调用者跨 ≥ 3 BF** | 分配到 `shared_foundations` | 0.90 - 0.99 |
| naming + import_proximity ≥ 2 个导入者 | 分配到命名提示的 BF | 0.80 - 0.90* |
| naming only (≥ 1.0) | 分配到命名提示的 BF | 0.70 - 0.80 |
| call_graph only（被多个 BF 调用） | 分配到 `shared_foundations` | 0.60 - 0.70 |
| 无任何信号 | 标记为 orphan | — |

> *confidence 与 Jaccard 导入相似度正相关

此步骤确保 `rate-limiting`、`error-handling`、`request-tracing`、`data-cleanup`、`database-management` 等无种子 BF 能被正确识别，同时 `shared_foundations` 中的类型/配置文件被自动归入基础层。

#### Phase 7: 分类（体系自适应）

根据 Step 1 选择的分类体系，对每个业务功能标注 `category`。

### 2.2 聚类评估

生成聚类报告 (`templates/kb-cluster-summary.md`)：
- 各阶段指标（种子数、实体数、调用图深度等）
- 加权平均置信度（目标 ≥ 0.80）
- 孤儿文件列表（未分配文件，目标 ≤ 3）
- 低置信度业务功能警告（confidence < 0.70）

**产出**: `knowledge-base/.manifest.yaml`
```yaml
schema_version: "2.0"

project_profile:
  type: <project-type>
  language: <language>
  framework: <framework>
  seed_strategy: <strategy>
  classification: <classification-system>
  detection_signals:
    - dimension: <dimension>
      signal: <detected-signal>

business_functions:
  - name: "<semantic-name>"
    category: <classification-label>
    confidence: <0.0-1.0>
    seeds:
      - type: <seed-type>
        source: "<file:line>"
        handler: "<file:symbol>"
    api_endpoints: []        # web-api 类型
    commands: []             # cli 类型
    page_routes: []          # frontend 类型
    public_exports: []       # library 类型
    core_symbols:
      functions: [<key-functions>]
      types: [<key-types>]
    primary_files:
      - <file-path>
    shared_files:
      - file: <file-path>
        role: "<role-description>"
        also_in: [<other-bf-names>]
    data_entities: [<entity-names>]
    depends_on:
      - business_function: <bf-name>
        reason: "<dependency-reason>"
    clustering_signals:
      seed: true|false
      call_graph: true|false
      import_proximity: true|false
      naming: true|false
    size:
      primary_files: <N>
      shared_files: <N>
      total_files: <N>
      kb_tokens_estimate: <N>

shared_foundations:
  - name: "shared-foundations"
    category: foundation
    primary_files: [<shared-type/config-files>]
    used_by: [<all-bf-names>]

generated_at: "<ISO-8601>"
generated_by: "devloop-reverse (v2 clustering)"
total_business_functions: <N>
total_files_analyzed: <N>
clustering_quality_score: <0.0-1.0>

migration_from:
  schema_version: "1.0"
  previous_manifest: "knowledge-base/.migrated-from-v1/manifest.yaml"
  migration_date: "<ISO-8601>"
```

**门禁 G1.1** — 业务功能覆盖率:
- `files_assigned_to_at_least_one_bf / total_src_files >= 0.90`
- `seeds_attributed / total_seeds >= 0.95`（web-api: API 端点; cli: 命令; generic: 文件）
- `orphan_files <= 3`

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.2 "Clustering complete: N business functions (score: X.XX)"
```

## Step 3: 并行业务功能深度分析 (1.2.3)

**目标**: 每个业务功能生成 5 个 KB 文件。

### 拆分策略（单业务功能 < 50K tokens）

- `primary_files >= 30` 或 `kb_tokens_estimate >= 50K`: 按子域拆分子业务功能
- 共享文件: 由主要归属的业务功能的 agent 负责分析，在 `shared_files` 中记录角色
- 每个 agent **只访问自己负责的业务功能相关代码**，使用 `codegraph_node` (MCP) 读取符号、API、依赖

按业务功能列表，通过 `dispatching-parallel-agents` 为每个业务功能分派独立 agent。

### 每个业务功能的 agent 任务

使用以下模板生成 5 个文件到 `knowledge-base/business-functions/<bf-name>/`：

| 文件 | 模板 | 核心内容 |
|------|------|---------|
| `overview.md` | `templates/kb-module-overview.md` | 职责、核心功能、对外接口摘要、依赖拓扑、共享文件角色 |
| `api.md` | `templates/kb-module-api.md` | API端点/命令/页面路由的完整签名、参数、返回值、异常（按项目类型自适应） |
| `data-model.md` | `templates/kb-module-data-model.md` | 数据结构、类型定义、实体所有权（owned/shared）、次要实体 |
| `business-logic.md` | `templates/kb-module-business-logic.md` | 核心流程、业务规则、副作用、共享文件角色声明 |
| `dependencies.md` | `templates/kb-module-dependencies.md` | 上下游依赖、跨BF依赖类型（call/import/entity_share/middleware_chain/route_mount等） |

### 防御标记

每个文件 frontmatter 必须包含：
```yaml
---
business_function: <bf-name>
category: <classification-label>
source: auto-generated
confidence: <0.0-1.0>       # 分析置信度
drift_score: <0.0-1.0>      # 初始为 0.0，后续由漂移检测更新
last_updated: <ISO-8601>
generated_from:              # 来源追溯
  - <source-file-paths>
---
```

`confidence` 判定规则：
- 0.9-1.0: API 签名/类型定义明确，代码结构清晰
- 0.7-0.9: 有一定推断成分（业务逻辑、隐含假设）
- < 0.7: 复杂逻辑、推断成分高，标记 low confidence

**门禁 G1.2** — KB 条目完整性:
- 每个 `knowledge-base/business-functions/<name>/` 下必须存在 `overview.md` + `api.md`
- 若 `knowledge-base/modules/` 目录存在（v1 遗留），标记为 stale

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.3 "BF analysis complete: N business functions, M KB files"
```

## Step 4: KB 汇总整合 (1.2.4)

**目标**: 从各业务功能 KB 条目生成项目级全局文档。

基于所有 `knowledge-base/business-functions/<name>/` 条目，生成：

```
knowledge-base/
├── architecture/
│   ├── overview.md        # 整体架构风格、层次结构（基于 BF 分类体系）
│   ├── components.md      # 业务功能拓扑、通信关系
│   └── data-flow.md       # 关键数据流路径（跨 BF）
├── apis/
│   └── internal.md        # 业务功能间 API/命令/路由总览
└── data-models/
    └── global.md          # 跨业务功能共享数据模型（基于 shared_foundations 去重）
```

汇总要求：
- 业务功能间引用使用 `[[bf-name]]` wiki link 语法
- 全局 API 文档按项目类型组织（web-api: 端点表; cli: 命令树; frontend: 路由图）
- 全局数据模型标注实体所有权（owned by BF vs shared）
- `shared_foundations` 不在全局数据模型中重复展开

**门禁 G1.3** — KB 一致性:
- 硬门禁: 所有 `[[bf-name]]` wiki link 目标存在
- 软门禁: 交叉引用语义匹配（LLM 评估）

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.4 "KB consolidation complete"
```

## Step 5: 需求反向推断 (1.2.5)

**目标**: 从汇总 KB 反推基线需求文档。

加载 `brainstorming` Skill，输入为：
- `knowledge-base/architecture/overview.md`
- `knowledge-base/apis/internal.md`
- `knowledge-base/data-models/global.md`
- 各业务功能 `overview.md`（摘要层）
- 聚类摘要 `knowledge-base/.manifest.yaml`（含各 BF 的 seeds 和 core_symbols）

产出到 `requirements/baseline/REQ-<NNN>-<slug>.md`：

```markdown
---
id: REQ-<NNN>
title: <标题>
module: <关联业务功能名>           # 旧字段保留向后兼容，值域扩展为 BF 名
business_function: <关联业务功能名> # 🆕 v2 字段
status: baseline
inferred_from:
  - <源文件路径>
confidence: <high|medium|low>
source: auto-inferred
usage_restriction: reference_only
created: <ISO-8601>
---

> ⚠️ **AUTO-INFERRED BASELINE** | confidence: <level>
> | source: code reverse-engineering
>
> This requirement was inferred from existing code implementation,
> NOT from original design documents. It may not reflect original design intent.
>
> **Usage restriction (`reference_only`)**: This baseline requirement serves
> as **documentation reference only**. It MUST NOT be used as a mandatory
> constraint when designing new features in Stage 3. If a new requirement
> conflicts with a baseline requirement, the baseline requirement gives way.

# <标题>

## 推断来源
...

## 功能描述
...

## 当前实现
...
```

### 基线需求规则

- **frontmatter 必须包含**: `source: auto-inferred`, `usage_restriction: reference_only`
- **用途限制**: `reference_only` = 仅供阶段二影响分析时作为背景知识阅读，**不得**作为阶段三设计方案的强制约束
- **冲突处理**: 新需求与基线需求冲突时，以新需求为准
- 每个 core/admin/command/page/public-api 类别的业务功能至少被 1 个基线需求覆盖（infrastructure/foundation 类别豁免）

**门禁 G1.4** — 需求反向覆盖率:
- `bfs_covered (core+admin+command+page+public-api) / total_target_bfs >= 0.90`
- `count(confidence == low) / count(all_req) <= 0.20`

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.5 "Baseline requirements created: N reqs"
```

## Step 6: 门禁校验 (1.2.6)

运行阶段一全部门禁：

```bash
.comet/scripts/devloop-guard check stage-1
```

必须看到 `ALL CHECKS PASSED` 或确认失败项可接受后方可继续。

**门禁 G1.0** — 项目感知有效性（🆕 v2）：
- [ ] `project_profile.type` 非空
- [ ] `seed_strategy` 与 `project_profile.type` 匹配
- [ ] `detection_signals` 维度 >= 2

**门禁 G1.1** — 业务功能覆盖率（v2 更新）：
- [ ] 文件覆盖率 >= 90%
- [ ] 种子归属率 >= 95%
- [ ] 孤儿文件 <= 3

**门禁 G1.2** — KB 条目完整性（v2 更新）：
- [ ] 每个业务功能有 `overview.md` + `api.md`
- [ ] v1 遗留 `modules/` 目录已标记 stale

**门禁 G1.3** — KB 一致性：
- [ ] wiki links 有效（硬门禁）
- [ ] 交叉引用语义匹配（软门禁）

**门禁 G1.4** — 需求反向覆盖率（v2 更新）：
- [ ] core/admin/command/page/public-api BF 覆盖率 >= 90%（排除 infrastructure/foundation）
- [ ] low confidence 基线需求 <= 20%

**门禁 G1.5** — 阶段一总门禁（v2 更新）：
- [ ] G1.0: 项目感知通过
- [ ] G1.2: KB 完整性通过
- [ ] G1.3.hard: KB 一致性硬门禁通过
- [ ] `clustering_quality_score >= 0.80`
- [ ] `orphan_files <= 3`
- [ ] 所有产物已 `git add`，无 merge conflict
- [ ] KB 总大小 > 0（防止空提交）

**软门禁处理**: 若 guard 输出 `SOFT_GATES_PENDING`，由编排层委托 LLM agent 逐项评估软门禁，结果写入 `.comet/gate-results/soft-<gate-id>.json`。

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G1.5 <pass|fail> "Stage-1 gates complete"
```

## Step 7: Git 提交 (1.2.7)

```bash
git checkout -b kb/update-$(date +%Y-%m-%d)
git add knowledge-base/ requirements/baseline/
git commit -m "[devloop] stage:1 action:reverse v2 complete:$(date +%Y-%m-%d)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**状态保存**:
```bash
.comet/scripts/comet-state complete-step 1 all
```

## 增量模式

当仅需更新部分业务功能时（漂移检测后或保鲜触发），指定 `--scope`：

```bash
# devloop-reverse --scope business-functions=user-authentication,jwt-token-management
```

增量模式下：
- Step 1 跳过（复用已有 `project_profile`）
- Step 2 仅对变更文件重新聚类（增量合并到已有 `business_functions` 列表）
- Step 3 仅分析指定业务功能
- Step 4 增量更新汇总文档（仅更新涉及部分的描述）
- Step 5 仅更新涉及业务功能的基线需求（已有需求标记 `updated`）
- Step 6/7 正常执行

## 向后兼容

### v1 模块目录迁移

若检测到 `knowledge-base/modules/` 存在且 `knowledge-base/.manifest.yaml` 为 v1 schema：

1. 将旧 manifest 备份到 `knowledge-base/.migrated-from-v1/manifest.yaml`
2. 将旧模块目录移动到 `knowledge-base/.migrated-from-v1/modules/`
3. 生成 v2 manifest（`business_functions` 从旧 `modules` 推导，`confidence` 标记为 0.70，附 `migrated: true`）
4. 后续运行中，若覆盖到相同业务功能，confidence 提升至聚类算法计算值

### 消费方兼容

读取 manifest 的脚本/工具按以下顺序查询：
1. 先查 `business_functions:`（v2）
2. 不存在则 fallback `modules:`（v1）
3. 两者都不存在则报错

## 复用清单

| 调用 | 用途 |
|------|------|
| `codegraph_explore` (MCP) | Step 1: 检测项目类型信号; Step 2: 种子收集、实体识别、调用图扩展 |
| `codegraph_callers` (MCP) | Step 2 Phase 3: 沿调用图向外扩展，查找实体操作者 |
| `codegraph_node` (MCP) | Step 3: 读取单个业务功能的符号、API、依赖 |
| `dispatching-parallel-agents` | Step 3: 并行分析多个业务功能（每个业务功能一个 agent） |
| `brainstorming` | Step 5: 从汇总 KB 反推基线需求 |
| `comet-state` (Phase 0) | 每步保存 checkpoint |
| `devloop-guard` (Phase 0) | Step 6: 运行门禁 G1.0-G1.5 |

## 防御原则

1. **每个 KB 条目** frontmatter 含 `source: auto-generated`, `business_function`, `category`, `confidence`, `drift_score`
2. **每个基线需求** 含 `source: auto-inferred`, `usage_restriction: reference_only`, `business_function` 字段
3. **单业务功能上下文** < 50K tokens（超大 BF 拆分子功能）
4. **来源可追溯**: 每个 KB 条目记录 `generated_from` 源代码文件列表
5. **交叉验证**: API 签名等关键信息对比 codegraph 实时查询结果，不一致时降 confidence
6. **共享文件透明**: 归属多个业务功能的文件在 manifest 和 KB 条目中显式标注角色和 `also_in` 列表
