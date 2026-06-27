# 01 — 总体设计 (High-Level Design)

## 1. 系统定名与愿景

**系统名称**: DevLoop — Closed-Loop Development System

**一句话**: 从现有代码中逆向提取知识库和需求文档，接受外部需求输入后自动结合知识库生成新需求，通过 Comet (OpenSpec + Superpowers) 模式驱动代码生成，验证通过后自动提交并回写需求状态，形成可持续运转的开发闭环。

---

## 2. 核心理念

### 2.1 知识驱动 (Knowledge-Driven)

传统开发流程中，需求文档和代码是分离的，文档容易过时。DevLoop 将**知识库 (Knowledge Base)** 作为一等公民：

- **逆向阶段**: 代码 → 结构化知识库 → 需求文档
- **正向阶段**: 需求文档 + 知识库 → 精准代码生成

知识库是连接两个方向的桥梁，也是 AI 理解项目的"长期记忆"。

### 2.2 门禁驱动 (Gate-Driven)

每个阶段、每个产物都有明确的质量门禁。参考 Comet 的 guard 机制，门禁不通过则**阻断流程**，不允许进入下一步。这确保自动化不会产生质量滑坡。

### 2.3 Git 原生 (Git-Native)

一切产物（知识库、需求文档、设计文档、计划、代码变更）都纳入 Git 管理。好处：

- 天然版本控制，可追溯、可回滚
- 分支策略支持并行需求开发
- commit message 天然形成变更日志
- PR/MR 流程支持人工 Review 介入

### 2.4 中断可恢复 (Resumable)

长流程必然面临中断（会话超时、上下文压缩、人工确认等待）。每一步都有**状态持久化**机制，恢复后从断点继续，无需重做。

---

## 3. 五阶段闭环架构

```
                    ┌──────────────────┐
                    │   知识库 (KB)     │
                    │   Git 管理        │
                    └───┬──────────┬───┘
                        ↑          ↓
              ① 逆向分析          ② 需求摄入
              Code→KB→Req         Ext+KB→Req
                        │          │
                        ↓          ↓
                    ┌───┴──────────┴───┐
                    │   需求文档 (Req)   │
                    │   Git 管理         │
                    └───┬──────────┬───┘
                        ↑          ↓
              ⑤ 回写完成          ③ 代码生成
              Req←Done            Req→Code
                        │          │
                        ↑          ↓
                    ┌───┴──────────┴───┐
                    │   代码变更 (Code)  │
                    │   Git 管理         │
                    └────────┬─────────┘
                             ↓
                        ④ 验证交付
                        Verify→Commit
```

### 3.1 阶段一：逆向分析 (Reverse Engineering)

**输入**: 现有代码仓库  
**输出**: 结构化知识库 + 基线需求文档  
**触发**: 系统初始化 / 代码大版本变更后 / 手动触发

**核心流程**:
1. 代码扫描 → 模块/组件识别
2. 逐模块深度分析 → API、数据模型、业务逻辑
3. 汇总为结构化知识库 (KB)
4. 从 KB 反向推断需求文档
5. 提交至 Git (`/knowledge-base/` + `/requirements/baseline/`)

**关键 Skill**:
| Skill | 作用 | 产物 | 来源 |
|-------|------|------|------|
| `codegraph_explore` (MCP) | 代码结构扫描 | 模块清单 + 符号依赖图 | ✅ 已有 |
| `dispatching-parallel-agents` | 并行分析多个模块 | 各模块 KB 条目 | ✅ 已有 |
| `brainstorming` | 从 KB 反推需求 | 基线需求文档 | ✅ 已有 |
| `devloop-reverse` | 阶段一总编排 | 完整 KB + 基线需求 | 🆕 新建 |

### 3.2 阶段二：需求摄入 (Requirements Intake)

**输入**: 外部需求（用户故事/功能请求/Bug报告）+ 知识库  
**输出**: 结构化需求文档（存入 Git）  
**触发**: 外部输入 / 阶段五回环 / 手动创建

**核心流程**:
1. 接收外部需求输入（支持多种格式）
2. 加载相关知识库上下文
3. 影响分析 (Impact Analysis) — 需求涉及哪些模块
4. 需求完善 — 补充技术细节、验收标准
5. 生成结构化需求文档
6. 提交至 Git (`/requirements/in-progress/`)

**关键 Skill**:
| Skill | 作用 | 产物 | 来源 |
|-------|------|------|------|
| `brainstorming` | 影响分析（提供 KB 上下文作为输入） | 影响分析报告 | ✅ 已有 |
| `openspec-propose` | 创建正式 Proposal | proposal.md + .comet.yaml | ✅ 已有 |
| `codegraph_explore` (MCP) | 索引式 KB 查询 | 相关模块 KB 条目 | ✅ 已有 |
| `devloop-intake` | 阶段二总编排 | 完整需求文档 | 🆕 新建 |

### 3.3 阶段三：代码生成 (Code Generation)

**输入**: 需求文档（从 Git 拉取）+ 知识库 + 现有代码  
**输出**: 代码变更（实现需求）  
**触发**: 阶段二完成 / 自动拉取未完成需求

**核心流程**:
1. 从 Git 拉取未完成需求列表
2. 选取优先级最高的需求
3. 加载知识库 + 需求文档作为上下文
4. 进入 Comet 标准流程: Design → Plan → Build
5. 产出代码变更

**关键 Skill**:
| Skill | 作用 | 产物 |
|-------|------|------|
| `brainstorming` | 设计方案 | design.md | ✅ 已有 |
| `writing-plans` | 创建实现计划 | plan.md, tasks.md | ✅ 已有 |
| `subagent-driven-development` | 执行实现（大需求） | 代码变更 + 双审查 | ✅ 已有 |
| `executing-plans` | 执行实现（小需求） | 代码变更 | ✅ 已有 |
| `test-driven-development` | TDD 保障（被 sdd 内部调用） | 测试用例 | ✅ 已有 |
| `devloop-build` | 阶段三总编排 | 完整代码变更 | 🆕 新建 |

### 3.4 阶段四：验证与交付 (Verification & Delivery)

**输入**: 代码变更  
**输出**: 已验证的提交 + 需求状态更新  
**触发**: 阶段三完成

**核心流程**:
1. 自动化验证（测试、lint、构建）
2. 代码审查（AI + 可选人工）
3. 修复验证中发现的问题
4. Commit 代码到项目 Git
5. 更新需求文档状态为 `completed`
6. 将需求文档移至 `/requirements/completed/`

**关键 Skill**:
| Skill | 作用 | 产物 |
|-------|------|------|
| `verification-before-completion` | 验证纪律（必须看到测试通过输出） | 验证结果 | ✅ 已有 |
| `code-review` (内置) | 代码质量审查 | Review 报告 | ✅ 已有 |
| `systematic-debugging` | 失败修复 | 根因分析 + 修复 | ✅ 已有 |
| `finishing-a-development-branch` | 收尾合并 | 合并/PR | ✅ 已有 |
| `devloop-verify` | 阶段四总编排 | 验证报告 + 需求状态更新 | 🆕 新建 |

### 3.5 阶段五：闭环回写 (Loop Back)

**输入**: 完成的需求记录  
**输出**: 触发新一轮需求处理  
**触发**: 阶段四完成

**核心流程**:
1. 扫描 `/requirements/in-progress/` 是否有待处理需求
2. 有 → 自动进入阶段二（需求已存在，直接进入阶段三）
3. 无 → 进入待命状态，监听外部需求输入
4. 对于全新需求 → 从阶段二开始

---

## 4. 关键架构决策 (ADR)

### ADR-1: 需求与代码同仓库

**决策**: 需求文档和代码在同一 Git 仓库中，通过目录隔离。

**理由**:
- 原子 commit：代码变更和需求状态更新可以在同一个 commit 中
- 简化工具链：一套 Git 操作，不需要跨仓库协调
- 分支自然映射：一个需求分支 = 一个 feature branch
- PR 中同时看到需求和技术实现

**目录结构**:
```
project-root/
├── knowledge-base/        # 知识库
│   ├── architecture/
│   ├── modules/
│   ├── apis/
│   └── data-models/
├── requirements/          # 需求文档
│   ├── baseline/          # 基线（从代码反向推断）
│   ├── in-progress/       # 进行中
│   └── completed/         # 已完成
├── src/                   # 源代码
├── openspec/              # Comet 规范
└── .comet/                # Comet 配置与状态
```

### ADR-2: 知识库采用 Markdown + Frontmatter

**决策**: 知识库条目使用 Markdown 格式，YAML frontmatter 携带结构化元数据。

**理由**:
- 人机可读，Git diff 友好
- AI 模型天然理解 Markdown
- Frontmatter 提供结构化查询能力（模块名、版本、依赖等）
- 无需额外数据库，降低复杂度

### ADR-3: 状态管理双保险

**决策**: 流程状态同时存储在 `.comet/loop-state.yaml`（快状态）和 Git commits（慢状态）。

**理由**:
- `loop-state.yaml` 提供快速状态查询和中断恢复
- Git commits 提供最终的真实状态（与代码同步）
- 两者交叉验证，防止状态漂移

### ADR-4: 确认策略分级，默认高保障

**决策**: 按需求类型和优先级分三级确认策略，**默认使用最高保障级别**。

**三级确认**:

| 级别 | 确认点 | 适用场景 |
|------|--------|---------|
| **Level 3** (默认) | 需求内容 + 设计方案 + 实现计划 + 验证失败 | `priority: high` / 安全相关 / 涉及 >3 模块 |
| **Level 2** | 设计方案 + 验证失败 | `priority: medium` / 常规 feature |
| **Level 1** (快速通道) | 仅验证失败 | `priority: low` bugfix / chore |

**核心原则**:
- 默认 Level 3，不能由系统自动降级
- 用户可主动降级任意需求，但安全相关和涉及 >5 模块的需求禁止降级
- 支持批量确认优化（多个需求同时到达确认点时）

> 详细设计见 `02-detailed-design.md` 第 12 节。

### ADR-5: 单开发者范围

**决策**: Phase 1 仅支持单个开发者串行使用 DevLoop。同一时间最多一个活跃需求。

**理由**:
- 大幅简化状态管理、锁、冲突解决的复杂性
- 单开发者是当前实际使用场景
- 设计上预留并发扩展点（见 `02-detailed-design.md` 第 17 节并发锁预留）

**扩展路径**: 当需要多人使用时，启用 `.comet/loop-lock.yaml`，将 `active_requirement` 升级为 `active_requirements[]`。`

---

## 5. Git 策略详解

### 5.1 分支策略

```
main
  ├── kb/update-2026-06-27          # 知识库更新
  ├── req/new-feature-xxx           # 新需求开发
  ├── req/fix-bug-123               # Bug 修复需求
  └── ...
```

- 知识库更新使用 `kb/*` 分支
- 需求开发使用 `req/*` 分支
- 完成后合并至 main

### 5.2 Commit 规范

```
[devloop] stage:<1-5> action:<action> target:<target>

示例:
[devloop] stage:1 action:reverse module:auth-service
[devloop] stage:2 action:intake req:add-oauth2-support
[devloop] stage:3 action:build req:add-oauth2-support
[devloop] stage:4 action:verify req:add-oauth2-support
[devloop] stage:5 action:complete req:add-oauth2-support
```

### 5.3 需求状态流转

```
draft → proposed → designed → planned → building → verifying → completed
                                                      ↓
                                                  blocked (异常)
```

状态由 Git 中需求文档的 frontmatter `status` 字段记录，同时反映在目录位置：
- `baseline/` = status: baseline
- `in-progress/` = status: proposed ~ verifying
- `completed/` = status: completed

---

## 6. 系统边界与外部接口

```
┌─────────────────────────────────────────────────────┐
│                    DevLoop 系统                       │
│                                                       │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Stage 1 │→│ Stage 2 │→│ Stage 3 │→│ Stage 4 │   │
│  │ 逆向分析│  │ 需求摄入│  │ 代码生成│  │ 验证交付│   │
│  └─────────┘  └─────────┘  └─────────┘  └────┬────┘   │
│       ↑                                      ↓        │
│       └────────── Stage 5 闭环 ──────────────┘        │
│                                                       │
│  外部接口:                                            │
│  ← 外部需求输入 (CLI / API / Webhook / IDE Plugin)    │
│  ← 人工确认/决策                                      │
│  → Git Push (需求 + 代码)                             │
│  → 通知 (CI/CD / Slack / Email)                       │
└─────────────────────────────────────────────────────┘
```

---

## 7. 技术栈

| 层面 | 选型 | 说明 |
|------|------|------|
| 编排引擎 | DevLoop Skill (薄编排层) | 6 个新建 Skill，~800 行总计，内部委托 14 个已有 Skill |
| 代码分析 | CodeGraph MCP | 符号级代码知识图谱，直接调用不包 Skill |
| AI 引擎 | Claude (via Claude Code) | 所有智能决策与生成 |
| 版本控制 | Git | 所有产物存储 |
| 状态存储 | YAML + Git | 流程状态持久化 |
| 模板 | Markdown + `{{ }}` 占位符 | LLM 原生填充，无需独立模板引擎 |
| 设计→计划→构建→验证→调试 | 已有 Superpowers Skill | brainstorming, writing-plans, sdd, tdd, code-review, systematic-debugging 等（不重写） |
| 门禁系统 | devloop-guard 脚本 | 新建，18 个门禁规则 |

---

## 8. 成功指标

| 指标 | 目标 | 测量方式 |
|------|------|---------|
| 逆向覆盖率 | >90% 模块被 KB 覆盖 | KB 条目数 vs 代码模块数 |
| 需求生成质量 | 人工修改率 <20% | diff 统计 |
| 代码生成通过率 | 一次验证通过率 >70% | 阶段四首次通过率 |
| 闭环周期 | 单个需求全流程 <2h | 时间戳差值 |
| 中断恢复率 | 100% 可恢复 | 中断测试 |
| KB 漂移控制 | 每周漂移 < 5% | drift_score 平均值 |

## 9. 范围与限制

**当前范围 (Phase 1)**:
- 单开发者，串行执行
- 单代码仓库
- 通过测试正确性保证质量（不要求环境一致性）
- 默认高保障确认策略（Level 3）

**明确排除**:
- 多人协作 / 并发需求处理（设计已预留扩展点）
- 跨仓库需求
- 非功能需求的自动逆向（性能/安全等需人工补充）
- 完全无人值守的自主运行（关键节点保留人工确认）
