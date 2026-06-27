---
name: devloop-reverse
description: "DevLoop 阶段一：逆向分析 — 从现有代码生成 KB 条目 + 基线需求。串联代码扫描→并行模块分析→KB汇总→需求反推→门禁→提交。"
---

# DevLoop Reverse — 阶段一：逆向分析

从现有代码库生成结构化的知识库（KB）和基线需求（baseline requirements）。
所有产物自动标记 `source: auto-generated`，基线需求标记 `usage_restriction: reference_only`。

## 触发条件

1. 新项目启动时，首次构建知识库
2. 收到 `/devloop reverse` 命令（全量或 `--scope <module>` 增量）
3. KB 漂移检测触发增量重建（`drift_score > 0.3`）
4. 手动触发保鲜：`/devloop kb-fresh --mode fix`

## 流程总览

```
Step 1: 代码扫描       → codegraph_explore (MCP)         → .manifest.yaml
Step 2: 并行模块分析   → dispatching-parallel-agents      → knowledge-base/modules/<name>/
       (每个 agent)    → codegraph_node (MCP)
Step 3: KB 汇总整合    → (编排层 LLM)                    → architecture/ + apis/ + data-models/
Step 4: 需求反向推断   → brainstorming                    → requirements/baseline/REQ-*.md
Step 5: 门禁校验       → devloop-guard check stage-1     → G1.1 - G1.5
Step 6: Git 提交       → git commit                       → [devloop] stage:1 action:reverse
```

每个 Step 结束后调用 `comet-state checkpoint` 保存断点，支持中断恢复。

## Step 1: 代码扫描

**目标**: 识别项目所有模块，生成模块清单。

使用 `codegraph_explore` (MCP) 扫描项目结构：
- 查询: "列出项目的所有顶层模块和组件，按目录/包分组"
- 识别模块类型: service / gateway / library / utility / ui / data / config / other
- 统计每个模块的文件数、依赖关系

**产出**: `knowledge-base/.manifest.yaml`
```yaml
modules:
  - name: <module-name>
    path: <source-directory>
    type: <module-type>
    files: <count>
    dependencies: [<dep-modules>]
    keywords: [<extracted-keywords>]
    entities: [<core-entities>]
    apis: [<public-api-summary>]
    total_tokens_estimate: <estimated>
    summary_tokens_estimate: <overview-only-estimate>
```

**门禁 G1.1** — 模块覆盖率: modules_analyzed / total_modules >= 0.8。
若未达标，提示用户补充手动指定的目录后重新扫描。

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.1 "Module manifest created: N modules found"
```

## Step 2: 并行模块深度分析

**目标**: 每个模块生成 5 个 KB 文件。

### 拆分策略（单模块 < 50K tokens）

- 小型项目（< 100 文件）: 按顶层目录拆分
- 中型项目（100-500 文件）: 按二级目录拆分
- 大型项目（500+ 文件）: 按功能域拆分
- 超大模块（> 100 文件）: 进一步拆分为子模块

按拆分后的模块列表，通过 `dispatching-parallel-agents` 为每个模块分派独立 agent。
每个 agent **只访问自己负责的模块代码**，使用 `codegraph_node` (MCP) 读取符号、API、依赖。

### 每个模块的 agent 任务

使用以下模板生成 5 个文件到 `knowledge-base/modules/<module-name>/`：

| 文件 | 模板 | 核心内容 |
|------|------|---------|
| `overview.md` | `templates/kb-module-overview.md` | 职责、核心功能、对外接口摘要、依赖拓扑 |
| `api.md` | `templates/kb-module-api.md` | 导出符号完整签名、参数、返回值、异常 |
| `data-model.md` | `templates/kb-module-data-model.md` | 数据结构、类型定义、实体关系 |
| `business-logic.md` | `templates/kb-module-business-logic.md` | 核心流程、业务规则、副作用 |
| `dependencies.md` | `templates/kb-module-dependencies.md` | 上下游依赖、耦合度、循环依赖检测 |

### 防御标记

每个文件 frontmatter 必须包含：
```yaml
---
module: <module-name>
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

**门禁 G1.2** — KB 条目完整性: 每个模块目录下必须存在 `overview.md` + `api.md`。
缺失模块标记后重新分析。

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.2 "Module analysis complete: N modules, M KB files"
```

## Step 3: KB 汇总整合

**目标**: 从各模块 KB 条目生成项目级全局文档。

基于所有 `knowledge-base/modules/<name>/` 条目，生成：

```
knowledge-base/
├── architecture/
│   ├── overview.md        # 整体架构风格、层次结构
│   ├── components.md      # 组件拓扑、通信关系
│   └── data-flow.md       # 关键数据流路径
├── apis/
│   └── internal.md        # 模块间内部 API 总览
└── data-models/
    └── global.md          # 跨模块共享数据模型
```

汇总要求：
- 模块间引用使用 `[[module-name]]` wiki link 语法
- 全局 API 文档列出各模块的核心接口及调用关系
- 全局数据模型去重跨模块的共享实体

**门禁 G1.3** — KB 一致性: 所有 `[[module-name]]` 引用目标存在，语义匹配。
硬门禁通过脚本检测引用有效性；软门禁通过 LLM 评估语义一致性。

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.3 "KB consolidation complete"
```

## Step 4: 需求反向推断

**目标**: 从汇总 KB 反推基线需求文档。

加载 `brainstorming` Skill，输入为：
- `knowledge-base/architecture/overview.md`
- `knowledge-base/apis/internal.md`
- `knowledge-base/data-models/global.md`
- 各模块 `overview.md`（摘要层）

产出到 `requirements/baseline/REQ-<NNN>-<slug>.md`：

```markdown
---
id: REQ-<NNN>
title: <标题>
module: <关联模块>
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
- 每个模块至少被 1 个基线需求覆盖

**门禁 G1.4** — 需求反向覆盖率:
- covered_modules / total_modules >= 0.9
- low confidence 需求占比 <= 20%

**状态保存**:
```bash
.comet/scripts/comet-state checkpoint 1 1.4 "Baseline requirements created: N reqs"
```

## Step 5: 门禁校验

运行阶段一全部门禁：

```bash
.comet/scripts/devloop-guard check stage-1
```

必须看到 `ALL CHECKS PASSED` 或确认失败项可接受后方可继续。

**门禁 G1.5** — 阶段一总门禁（汇总判定）：
- [ ] G1.1: 模块覆盖率 >= 80%
- [ ] G1.2: 所有模块 KB 条目完整（至少 overview.md + api.md）
- [ ] G1.3: KB 一致性验证通过（wiki links 有效 + 语义匹配）
- [ ] G1.4: 需求覆盖率 >= 90%，low confidence <= 20%
- [ ] 所有产物已 `git add`，无 merge conflict
- [ ] KB 总大小 > 0（防止空提交）

**软门禁处理**: 若 guard 输出 `SOFT_GATES_PENDING`，由编排层委托 LLM agent 逐项评估软门禁，结果写入 `.comet/gate-results/soft-<gate-id>.json`。

**状态保存**:
```bash
.comet/scripts/comet-state gate-result G1.5 <pass|fail> "Stage-1 gates complete"
```

## Step 6: Git 提交

```bash
git checkout -b kb/update-$(date +%Y-%m-%d)
git add knowledge-base/ requirements/baseline/
git commit -m "[devloop] stage:1 action:reverse complete:$(date +%Y-%m-%d)"

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**状态保存**:
```bash
.comet/scripts/comet-state complete-step 1 all
```

## 增量模式

当仅需更新部分模块时（漂移检测后或保鲜触发），指定 `--scope`：

```bash
# devloop-reverse --scope modules=auth-service,api-gateway
```

增量模式下：
- Step 1 跳过（复用已有 manifest）
- Step 2 仅分析指定模块
- Step 3 增量更新汇总文档（仅更新涉及部分的描述）
- Step 4 仅更新涉及模块的基线需求（已有需求标记 `updated`）
- Step 5/6 正常执行

## 复用清单

| 调用 | 用途 |
|------|------|
| `codegraph_explore` (MCP) | 扫描项目结构，生成模块清单 |
| `codegraph_node` (MCP) | 读取单模块的符号、API、依赖 |
| `dispatching-parallel-agents` | 并行分析多个模块（每个模块一个 agent） |
| `brainstorming` | 从汇总 KB 反推基线需求 |
| `comet-state` (Phase 0) | 每步保存 checkpoint |
| `devloop-guard` (Phase 0) | 运行门禁 G1.1-G1.5 |

## 防御原则

1. **每个 KB 条目** frontmatter 含 `source: auto-generated`, `confidence`, `drift_score`
2. **每个基线需求** 含 `source: auto-inferred`, `usage_restriction: reference_only`
3. **单模块上下文** < 50K tokens（超大模块拆分子模块）
4. **来源可追溯**: 每个 KB 条目记录 `generated_from` 源代码文件列表
5. **交叉验证**: API 签名等关键信息对比 codegraph 实时查询结果，不一致时降 confidence
