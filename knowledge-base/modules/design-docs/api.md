---
module: design-docs
source: auto-generated
confidence: 0.95
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - design/00-README.md
  - design/01-overview.md
  - design/02-detailed-design.md
  - design/03-flowcharts.md
  - design/05-implementation-tasks.md
---

# design-docs — API 文档

设计文档本身作为 DevLoop 系统的**配置层**，其"API"定义为它所声明的接口规范、编排流程、数据格式和门禁规则。所有新建 Skill、脚本、模板均以这些 API 规范为契约。

---

## 1. 编排 Skill 接口

### 1.1 devloop-reverse（阶段一：逆向分析）

**输入**: 项目根目录路径
**输出**: 结构化知识库 + 基线需求文档（提交至 Git）
**门禁**: G1.1-G1.5
**内部委托链**:
```
devloop-reverse
├── codegraph_explore (MCP)       — 代码扫描（1.1）
├── dispatching-parallel-agents    — 并行模块分析（1.2），每个 agent 内部用 codegraph_node
├── (编排层逻辑)                    — KB 汇总整合（1.3）
├── brainstorming                  — 需求反向推断（1.4）
├── devloop-guard                  — 门禁校验（1.5）
├── git (Bash)                     — 提交至 Git（1.6）
└── comet-state                    — 状态管理
```

### 1.2 devloop-intake（阶段二：需求摄入）

**输入**: 外部需求（用户故事/功能请求/Bug报告）+ 知识库
**输出**: 结构化需求文档（存入 `requirements/in-progress/`）
**门禁**: G2.1-G2.5
**内部委托链**:
```
devloop-intake
├── (编排层逻辑)                    — 输入解析（2.1）
├── codegraph_explore (MCP)        — KB 上下文加载（2.2）
├── (索引逻辑)                     — 关键词匹配 + 索引查询
├── brainstorming                  — 影响分析（2.3）
├── (NFR 评估清单)                 — 非功能需求强制评估
├── brainstorming + openspec-propose — 需求文档生成（2.4）
├── devloop-guard                  — 门禁校验（2.5）
├── git (Bash)                     — 提交
└── comet-state                    — 状态管理
```

### 1.3 devloop-build（阶段三：代码生成）

**输入**: 需求文档（从 Git 拉取）+ 知识库 + 现有代码
**输出**: 代码变更（实现需求）
**门禁**: G3.1-G3.4
**内部委托链**:
```
devloop-build
├── (编排层逻辑)                    — 拉取需求 + 优先级排序（3.1-3.2）
├── brainstorming                  — 设计方案（3.3）
├── writing-plans                  — 创建实现计划（3.4）
├── subagent-driven-development    — 执行实现（大需求，3.5）
├── executing-plans                — 执行实现（小需求 < 5 tasks，3.5）
├── using-git-worktrees            — 隔离工作空间
├── devloop-guard                  — 门禁校验
├── git (Bash)                     — 提交
└── comet-state                    — 状态管理
```

### 1.4 devloop-verify（阶段四：验证交付）

**输入**: 代码变更
**输出**: 已验证的提交 + 需求状态更新
**门禁**: G4.1-G4.4
**内部委托链**:
```
devloop-verify
├── verification-before-completion — 验证纪律（4.1）
├── code-review (内置)             — 代码质量审查（4.2）
├── systematic-debugging           — 失败修复（4.3）
├── finishing-a-development-branch — 收尾合并（4.5）
├── (编排层逻辑)                    — 更新需求状态（4.6-4.7）
├── devloop-guard                  — 门禁校验
├── git (Bash)                     — 提交
└── comet-state                    — 状态管理
```

### 1.5 devloop-loop（阶段五：闭环回写）

**输入**: 完成的需求记录
**输出**: 触发新一轮需求处理
**门禁**: G5.1
**内部委托链**:
```
devloop-loop
├── (编排层逻辑)                    — 扫描待处理需求（5.1）
├── (编排层逻辑)                    — 优先级排序 + 路由（5.2）
├── (编排层逻辑)                    — 监听模式（5.3）
└── comet-state                    — 状态管理
```

### 1.6 devloop-resume（中断恢复）

**输入**: loop-state.yaml（含有活跃状态）
**输出**: 恢复后的对话上下文 + 继续执行
**流程**:
```
devloop-resume
├── 读取 loop-state.yaml
├── 显示中断位置 + 上下文摘要
├── 运行 comet-state check <name> <phase> --recover
├── 按 Recovery action 继续（resume_from_step / replay_missing / restart_stage）
└── 更新 checkpoint
```

---

## 2. 脚本接口规范

### 2.1 comet-state（状态管理）

| 操作 | 命令 | 说明 |
|------|------|------|
| init | `comet-state init <project>` | 初始化状态文件 |
| set | `comet-state set <field> <value>` | 设置状态字段 |
| get | `comet-state get <field>` | 读取状态字段 |
| check | `comet-state check <name> <phase> --recover` | 中断恢复 |
| scale | `comet-state scale <name>` | 确定验证级别 |

**输出格式**: YAML
**存储文件**: `.comet/loop-state.yaml`

### 2.2 devloop-guard（门禁执行引擎）

| 操作 | 命令 | 说明 |
|------|------|------|
| list | `devloop-guard list` | 列出所有门禁规则 |
| run | `devloop-guard run <gate-id>` | 运行单个门禁 |
| run-all | `devloop-guard run-all --stage <1-5>` | 运行阶段全套门禁 |
| status | `devloop-guard status` | 查询当前门禁状态 |

**执行模型**: 两阶段 — 先硬门禁（脚本自动判定），全部通过后软门禁（LLM 语义评估）
**配置文件**: `templates/guard-config.yaml`

### 2.3 kb-drift-check（KB 漂移检测）

| 操作 | 命令 | 说明 |
|------|------|------|
| check | `kb-drift-check --scope <module|all>` | 检测漂移 |
| report | `kb-drift-check --report` | 生成漂移报告 |

**输出**: `knowledge-base/.drift-report.yaml`

### 2.4 kb-freshness-trigger（KB 保鲜触发）

| 操作 | 命令 | 说明 |
|------|------|------|
| run | `kb-freshness-trigger --mode <report|fix>` | 执行保鲜检查 |
| scope | `--scope <module|all>` | 指定范围 |

### 2.5 kb-trust-update（KB 信任级别评估）

| 操作 | 命令 | 说明 |
|------|------|------|
| update | `kb-trust-update --module <name>` | 更新模块信任级别 |

### 2.6 confirmation-queue（确认队列管理）

| 操作 | 命令 | 说明 |
|------|------|------|
| list | `confirmation-queue list` | 列出待处理确认 |
| confirm | `confirmation-queue confirm <id>` | 确认某项 |
| reject | `confirmation-queue reject <id>` | 拒绝某项 |

---

## 3. 知识库 KB 条目格式规范

每个 KB 条目为 Markdown 文件，带 YAML frontmatter。

### Frontmatter 必填字段

```yaml
---
module: <module-name>           # 模块名
source: auto-generated           # auto-generated | human-reviewed | hybrid
confidence: <0.0-1.0>           # 可信度
drift_score: <0.0-1.0>          # 漂移度
last_updated: <ISO 8601>        # 最后更新时间
generated_from:                  # 来源追溯
  - src/auth/login.ts
---
```

### 条目类型

| 文件名 | 内容 | 模板 |
|--------|------|------|
| overview.md | 模块概述、职责、关键指标 | kb-module-overview.md |
| api.md | 对外接口 API 文档 | kb-module-api.md |
| data-model.md | 数据模型/类型定义 | kb-module-data-model.md |
| business-logic.md | 核心业务逻辑描述 | kb-module-business-logic.md |
| dependencies.md | 模块依赖关系 | kb-module-dependencies.md |

### KB Manifest 清单

```yaml
# knowledge-base/.manifest.yaml
modules:
  - name: <module-name>
    path: src/<module>/
    type: service | gateway | library | utility | ui | data | config
    files: <count>
    dependencies: [<dep1>, <dep2>]
    keywords: [<kw1>, <kw2>]           # 用于索引式加载
    entities: [<entity1>, <entity2>]   # 数据实体
    apis: [<api1>, <api2>]             # API 接口
    total_tokens_estimate: <int>       # KB 条目总 token 估算
    summary_tokens_estimate: <int>     # overview.md 单独 token 估算
```

---

## 4. 需求文档格式规范

### 统一需求文档 frontmatter

```yaml
---
id: REQ-001
title: 核心认证功能
module: auth-service
type: feature | bugfix | enhancement | refactor | chore
priority: high | medium | low
status: proposed | designed | planned | building | built | verifying | completed | blocked
kb_context:
  - knowledge-base/modules/auth-service/
  - knowledge-base/modules/user-service/
acceptance_criteria:
  - "POST /api/auth/login 接口实现"
  - "JWT Token 正确生成和验证"
created: <ISO 8601>
updated: <ISO 8601>
---
```

### 基线需求特殊字段

基线需求（从代码反向推断）额外携带：

```yaml
source: auto-inferred          # 标识来源
usage_restriction: reference_only  # 用途限制
inferred_from:
  - src/auth/login.ts          # 推断来源
confidence: high | medium | low
```

**核心规则**: `usage_restriction: reference_only` 标记的基线需求**只能作为背景参考**，禁止作为阶段三新功能设计的强制约束。门禁 G3.2 专门检查此规则。

---

## 5. 确认级别接口

| 级别 | 名称 | 确认点 | 适用场景 |
|------|------|--------|---------|
| Level 3 | 高保障 | 需求内容 + 设计方案 + 实现计划 + 验证失败 | priority:high, 涉及 >3 模块, 安全相关 |
| Level 2 | 标准保障 | 设计方案 + 验证失败 | priority:medium, affected <= 3 |
| Level 1 | 快速通道 | 仅验证失败 | priority:low bugfix, chore |

---

## 6. Commit 规范

```
[devloop] stage:<1-5> action:<action> target:<target>
[devloop] stage:1 action:reverse module:auth-service
[devloop] stage:2 action:intake req:add-oauth2-support
```
