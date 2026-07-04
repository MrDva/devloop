# 05 — DevLoop 实现任务清单 (v3: 防护加固)

> **变更记录**:
> - v1: 15 个新 Skill
> - v2: 缩减为 6 个编排 Skill + 6 个脚本，全部委托已有 Skill
> - v3 (当前): 门禁硬/软分离（防止空洞阻断）、基线需求用途限制（防止低质逆向污染下游）、阶段二插入 NFR 强制评估步骤（防止非功能需求断层）
> **使用方式**: 每个任务直接交给 Claude Code 执行，运行验证命令确认通过后进入下一个。

---

## 复用 vs 新建 总览

### 复用的已有 Skill（14 个，无需修改）

| 已有 Skill | 被哪个 DevLoop Skill 调用 | 调用阶段 |
|-----------|--------------------------|---------|
| `dispatching-parallel-agents` | devloop-reverse | 阶段一：并行分析多个业务功能 |
| `brainstorming` | devloop-reverse, devloop-intake, devloop-build | 阶段一（反推需求）、阶段二（影响分析）、阶段三（方案设计） |
| `openspec-propose` | devloop-intake | 阶段二：创建 OpenSpec Change |
| `writing-plans` | devloop-build | 阶段三：创建实现计划 |
| `subagent-driven-development` | devloop-build | 阶段三：执行实现（大需求） |
| `executing-plans` | devloop-build | 阶段三：执行实现（小需求） |
| `test-driven-development` | （被 subagent-driven-development 内部调用） | 阶段三：每个 task 的 TDD |
| `requesting-code-review` | （被 subagent-driven-development 内部调用） | 阶段三：每个 task 的双审查 |
| `verification-before-completion` | devloop-verify | 阶段四：验证纪律 |
| `code-review` (内置) | devloop-verify | 阶段四：代码审查 |
| `systematic-debugging` | devloop-verify | 阶段四：修复失败 |
| `finishing-a-development-branch` | devloop-verify | 阶段四：收尾合并 |
| `using-git-worktrees` | devloop-build | 阶段三：隔离工作空间 |
| `using-superpowers` | （启动时自动加载） | 元技能 |

### 已有 MCP 工具（直接调用，不包 Skill）

| MCP 工具 | 被哪个 DevLoop Skill 调用 | 用途 |
|---------|--------------------------|------|
| `codegraph_explore` | devloop-reverse, devloop-intake | 代码扫描、模块识别、KB索引查询 |
| `codegraph_node` | devloop-reverse | 单文件/符号级代码读取 |

### 新建的（本次实现）

| 类型 | 名称 | 行数估算 | 说明 |
|------|------|---------|------|
| **编排 Skill** | `devloop-reverse` | ~150 行 | 阶段一总编排：设KB上下文 → 委托已有Skill → 记录状态 |
| **编排 Skill** | `devloop-intake` | ~180 行 | 阶段二总编排：解析输入 → 查索引 → NFR评估 → 委托brainstorming → 创建需求 |
| **编排 Skill** | `devloop-build` | ~150 行 | 阶段三总编排：拉需求 → 委托brainstorming/writing-plans/sdd |
| **编排 Skill** | `devloop-verify` | ~120 行 | 阶段四总编排：委托verification-before-completion/code-review/debugging |
| **编排 Skill** | `devloop-loop` | ~80 行 | 阶段五总编排：扫描队列 → 路由 |
| **编排 Skill** | `devloop-resume` | ~80 行 | 中断恢复：读状态 → 回上下文 → 继续 |
| **脚本** | `comet-state` | ~200 行 | 状态管理（YAML读写） |
| **脚本** | `devloop-guard` | ~200 行 | 门禁执行引擎（硬/软分离，硬门禁脚本判定 + 软门禁待办清单输出） |
| **脚本** | `kb-drift-check` | ~100 行 | KB漂移检测 |
| **脚本** | `kb-freshness-trigger` | ~120 行 | KB保鲜触发 |
| **脚本** | `kb-trust-update` | ~80 行 | KB信任级别评估 |
| **脚本** | `confirmation-queue` | ~100 行 | 确认队列管理 |
| **模板** | 10 个 `.md` 文件 | ~50 行/个 | 产出物格式定义（含 NFR 检查清单） |

---

## Phase 0: 基础建设（5 个任务）

> **目标**: 目录、模板、state/guard 脚本就绪。
> **完成标志**: `comet-state init test` 和 `devloop-guard list` 正常运行。

---

### T0.0 — 环境检查

**目标**: 确认依赖工具可用，缺失则提示安装。

**验证**:
```bash
# 必需
git --version          || echo "❌ 需要 git"
# 推荐（脚本使用）
which yq  && echo "✓ yq"  || echo "⚠️ yq 未安装，comet-state 将降级为 grep/sed 模式"
which jq  && echo "✓ jq"  || echo "⚠️ jq 未安装"
# MCP 工具
test -d .codegraph/    && echo "✓ codegraph 已索引" || echo "⚠️ .codegraph/ 不存在，阶段一需要"
# 已有 Skill
test -d .claude/skills/brainstorming && echo "✓ brainstorming" || echo "❌ 缺少 brainstorming"
test -d .claude/skills/writing-plans && echo "✓ writing-plans" || echo "❌ 缺少 writing-plans"
test -d .claude/skills/subagent-driven-development && echo "✓ sdd" || echo "❌ 缺少 sdd"
echo "✅ T0.0 PASSED"  # 警告不阻塞，缺失项在后续任务中处理
```

---

### T0.1 — 创建目录结构 + 模板文件

**目标**: 一次性创建所有目录和 10 个模板（含基线需求用途限制声明、NFR 检查清单）。

**实现**:
1. 创建目录树: `knowledge-base/{architecture,modules,apis,data-models}`, `requirements/{baseline,in-progress,completed}`, `templates/`, `.comet/scripts/`, `.comet/gate-results/`
2. 创建 10 个模板文件（每个带 frontmatter + `{{ }}` 占位符 + 底部变量清单）
3. 创建 `knowledge-base/.manifest.yaml`（初始 `modules: []`）
4. **基线需求用途限制**: `templates/requirement-doc.md` 必须包含基线模式的条件块 — 当 `status == baseline` 时，渲染醒目的 `⚠️ AUTO-INFERRED BASELINE` 警告、`source: auto-inferred`、`usage_restriction: reference_only` 声明及含义

**模板清单**:
```
templates/
├── requirement-input.md         # 用户提交需求时填写
├── requirement-doc.md           # 阶段二产出：统一需求文档（含基线模式警告块）
├── kb-module-overview.md        # 模块概述
├── kb-module-api.md             # API 文档
├── kb-module-data-model.md      # 数据模型
├── kb-module-business-logic.md  # 业务逻辑
├── kb-module-dependencies.md    # 依赖关系
├── impact-analysis.md           # 影响分析报告
├── nfr-checklist.md             🆕 非功能需求强制评估清单
├── verification-checklist.md    # 验证检查清单
└── kb-manifest.yaml             # KB 清单模板
```

**验证**:
```bash
for d in knowledge-base/{architecture,modules,apis,data-models} \
         requirements/{baseline,in-progress,completed} templates .comet/scripts .comet/gate-results; do
  test -d "$d" || { echo "FAIL: $d missing"; exit 1; }
done
for f in requirement-input requirement-doc kb-module-overview kb-module-api \
         kb-module-data-model kb-module-business-logic kb-module-dependencies \
         impact-analysis nfr-checklist verification-checklist; do
  test -s "templates/${f}.md" || { echo "FAIL: templates/${f}.md"; exit 1; }
  grep -q '{{.*}}' "templates/${f}.md" || { echo "FAIL: ${f}.md missing variables"; exit 1; }
done
# 基线需求模板必须包含用途限制声明
grep -q 'AUTO-INFERRED BASELINE\|usage_restriction.*reference_only' templates/requirement-doc.md \
  && echo "✅ 基线需求用途限制已嵌入模板" \
  || { echo "❌ requirement-doc.md 缺少基线用途限制声明"; exit 1; }
echo "✅ T0.1 PASSED"
```

---

### T0.2 — 实现 comet-state 脚本

**目标**: 状态管理脚本，支持 `init`/`set`/`get`/`checkpoint`/`complete-step`/`gate-result`/`check --recover`/`scale`。

**实现**:
1. 创建 `.comet/scripts/comet-state`（Bash，优先使用 `yq` 操作 YAML，不可用时降级为 grep/sed）
2. 创建初始 `loop-state.yaml` 模板

**子命令**:
```
comet-state init <change-name>           # 初始化状态文件
comet-state set <key> <value>            # 设置字段
comet-state get <key>                    # 读取字段
comet-state checkpoint <stage> <step> "<desc>"  # 保存断点
comet-state complete-step <stage> <step> # 标记步骤完成
comet-state gate-result <gate-id> <pass|fail> [msg]  # 记录门禁
comet-state add-pending <type> <req> "<msg>"        # 添加确认项
comet-state check <name> <phase> --recover          # 恢复检查
comet-state scale <name>                 # 显示状态摘要
```

**验证**:
```bash
chmod +x .comet/scripts/comet-state
.comet/scripts/comet-state init test
.comet/scripts/comet-state set current_stage 1
test "$(.comet/scripts/comet-state get current_stage)" = "1" \
  && echo "✅ T0.2 PASSED" || echo "❌ T0.2 FAILED"
```

---

### T0.3 — 实现 devloop-guard 脚本 + 门禁配置

**目标**: 门禁执行引擎 + 18 个门禁的完整配置（硬/软门禁分离）。

**实现**:
1. 创建 `.comet/scripts/devloop-guard`（Bash）
2. 创建 `.comet/guard-config.yaml`（18 个门禁，来自设计文档 7.3 节）
3. **硬/软门禁分离**: 每个门禁声明 `type: hard | soft | mixed`
   - **hard**: 脚本自动判定（`test -f`、`grep -c`、退出码、字段计数）→ 可 BLOCK
   - **软门禁判断规则**: 脚本先跑硬门禁，硬门禁全部通过后输出 `SOFT_GATES_PENDING` 信号；编排 Skill 收到此信号后委托 LLM agent 评估软门禁；LLM 评估结果以结构化 JSON 写入 `.comet/gate-results/soft-<gate-id>.json`
   - **mixed**: 硬门禁脚本判定 + 软门禁 LLM 判定（硬=BLOCK，软=WARN）
   - **软门禁核心约束**: soft 类型只能 WARN 不阻断
4. 支持四种 `on_fail` 策略: `block_and_retry`, `warn_and_continue`, `reject`, `pause_for_human`
5. 输出格式: `[PASS]`/`[WARN]`/`[FAIL]` + 详情，最终必须输出 `ALL CHECKS PASSED` 或列出失败项
6. **软门禁结果文件**: 脚本输出软门禁检查清单到 `.comet/gate-results/soft-gates-pending.json`，编排 Skill 读取后逐个委托 LLM 评估

**门禁清单**（18 个，含硬/软分类）:
```
G1.1 模块覆盖率 [hard]     G1.2 KB条目完整性 [hard]     G1.3 KB内部一致性 [mixed]
G1.4 需求反向覆盖率 [hard]  G1.5 阶段一总门禁 [hard]
G2.1 输入有效性 [hard]      G2.2 KB上下文相关性 [soft]    G2.3 影响分析完整性 [mixed]
G2.4 需求文档结构 [hard]     G2.5 阶段二总门禁 [hard]
G3.1 需求可构建性 [hard]    G3.2 Design阶段质量 [mixed]    G3.3 Plan阶段质量 [mixed]
G3.4 Build阶段质量 [hard]
G4.1 自动化验证 [hard]      G4.2 代码审查 [mixed]         G4.3 修复循环限制 [hard]
G4.4 阶段四总门禁 [hard]
G5.1 闭环状态一致性 [hard]
```

**验证**:
```bash
chmod +x .comet/scripts/devloop-guard
# 检查所有门禁已定义
.comet/scripts/devloop-guard list | grep -c "G[1-5]" | xargs -I{} test {} -ge 18 \
  && echo "✅ 门禁数量正确"
# 检查硬/软分类
.comet/scripts/devloop-guard list --type hard | grep -c "G" | xargs -I{} test {} -ge 12 \
  && echo "✅ 硬门禁 >= 12"
.comet/scripts/devloop-guard list --type soft,mixed | grep -c "G" | xargs -I{} test {} -ge 6 \
  && echo "✅ 含软门禁 >= 6"
# 验证软门禁禁止使用 BLOCK 级别阻断
.comet/scripts/devloop-guard validate-config  # 确保 soft 门禁 on_fail != block_and_retry
echo "✅ T0.3 PASSED"
```

---

### T0.4 — 创建 devloop-resume Skill

**目标**: 中断恢复 Skill。会话重启时读取 `loop-state.yaml` 回当前进度，处理待确认项，从 checkpoint 继续。

**实现**:
1. 创建 `.claude/skills/devloop-resume/SKILL.md`
2. 内部逻辑: 读 `loop-state.yaml` → 展示摘要 → 处理 pending_confirmations → 从 current_step 继续
3. 调用 `comet-state check --recover` 获取恢复动作
4. 快速上下文重建: 读 proposal.md + KB + memory 文件

**复用**: `comet-state` 脚本（T0.2）

**验证**:
```bash
test -s .claude/skills/devloop-resume/SKILL.md \
  && grep -q 'loop-state.yaml\|checkpoint\|恢复' .claude/skills/devloop-resume/SKILL.md \
  && echo "✅ T0.4 PASSED" || echo "❌ T0.4 FAILED"
```

---

## Phase 1: 阶段一 — 逆向分析（3 个任务）

> **目标**: 从现有代码生成 KB 和基线需求。
> **完成标志**: `knowledge-base/modules/` 下有 >=1 个模块的完整 KB 条目，`requirements/baseline/` 下有基线需求。

---

### T1.1 — 创建 devloop-reverse Skill (v2)

**目标**: 阶段一编排 Skill。串联项目感知→多信号聚类→并行业务功能分析→KB汇总→需求反推→门禁→提交。

**复用**:
| 调用 | 用途 |
|------|------|
| `codegraph_explore` (MCP) | 1.2.1 项目感知 + 1.2.2 多信号聚类各阶段 |
| `codegraph_callers` (MCP) | Phase 3 调用图扩展，Phase 2 实体锚点 |
| `codegraph_node` (MCP) | 读取单业务功能的符号、API、依赖 |
| `dispatching-parallel-agents` | 并行分析多个业务功能（每个功能一个 agent） |
| `brainstorming` | 从汇总 KB 反推基线需求 |
| `comet-state` (T0.2) | 每步保存 checkpoint |
| `devloop-guard` (T0.3) | 运行门禁 G1.0-G1.5 |

**新建逻辑**（薄编排层，~200 行）:
- **1.2.1 项目感知**: 检测项目类型（web-api/cli/frontend/library/generic），选择种子策略和分类体系
- **1.2.2 多信号聚类**: Phase 1-7 算法编排（种子收集→实体锚点→调用图→导入邻近→命名→投票合并→分类）
  - 聚类算法不是独立脚本——它是 LLM 驱动的 `codegraph_explore`/`codegraph_callers` 调用序列
  - 加权投票阈值可在 `guard-config.yaml` 配置
- 业务功能拆分策略（单功能 < 50K tokens）
- KB 条目 frontmatter 防御标记注入（`source: auto-generated`, `confidence`, `drift_score`, `clustering_signals`）
- KB 汇总整合（各业务功能条目 → architecture/apis/data-models 全局文档 + 依赖图）
- v1→v2 迁移编排（检测旧 manifest → 归档 → 聚类 → 更新需求引用 → 生成迁移报告）
- Git commit: `[devloop] stage:1 action:reverse`

**产出**: `.claude/skills/devloop-reverse/SKILL.md`

**验证**:
```bash
test -s .claude/skills/devloop-reverse/SKILL.md \
  && grep -q 'codegraph_explore\|dispatching-parallel-agents\|brainstorming' \
     .claude/skills/devloop-reverse/SKILL.md \
  && grep -q 'project_profile\|business_functions\|clustering' \
     .claude/skills/devloop-reverse/SKILL.md \
  && echo "✅ T1.1 PASSED" || echo "❌ T1.1 FAILED"
```

---

### T1.2 — 运行阶段一（首次逆向分析 v2）

**目标**: 在当前项目上执行 `devloop-reverse`，产出完整 KB（v2 格式）+ 基线需求。

**实现**:
1. 加载 `devloop-reverse` Skill
2. 对当前项目执行全量逆向分析（1.2.1 项目感知 → 1.2.2 聚类 → 1.2.3 功能分析）
3. 每个业务功能产出 5 个 KB 文件到 `knowledge-base/business-functions/<bf-name>/`
4. 汇总产出 architecture/apis/data-models 全局文档 + 业务功能依赖图
5. 反推基线需求到 `requirements/baseline/`
6. 每个产物 frontmatter 包含防御标记 + business_function 引用

**验证**:
```bash
BFS=$(ls knowledge-base/business-functions/ 2>/dev/null | wc -l)
REQS=$(ls requirements/baseline/REQ-*.md 2>/dev/null | wc -l)
echo "业务功能数: $BFS, 基线需求数: $REQS"
test "$BFS" -gt 0 && test "$REQS" -gt 0 \
  && echo "✅ T1.2 PASSED" || echo "❌ T1.2 FAILED"
# 检查 v2 格式
grep -q 'business_functions:' knowledge-base/.manifest.yaml \
  && echo "  ✓ v2 manifest 格式"
grep -q 'project_profile:' knowledge-base/.manifest.yaml \
  && echo "  ✓ project_profile 存在"
```

---

### T1.3 — 门禁校验 + 人工抽检

**目标**: 运行阶段一门禁，人工抽检低 confidence 条目。

**实现**:
1. 运行 `devloop-guard check stage-1`
2. 检查 `confidence: low` 的需求比例不超过 20%
3. 人工抽检: 选 1-2 个 KB 条目，对比真实代码验证准确性
4. 如门禁全部通过且抽检合格 → Git commit

**验证**:
```bash
.comet/scripts/devloop-guard check stage-1
# 检查低置信度比例
LOW=$(grep -r 'confidence: low' requirements/baseline/ | wc -l)
TOTAL=$(ls requirements/baseline/REQ-*.md | wc -l)
echo "低置信度需求: $LOW/$TOTAL"
echo "✅ T1.3 PASSED: 阶段一门禁完成"
```

---

## Phase 2: 阶段二 — 需求摄入（3 个任务）

> **目标**: 接收外部需求输入，结合 KB 生成结构化需求文档。
> **完成标志**: 输入一个需求描述，成功产出 `requirements/in-progress/REQ-*.md` + OpenSpec Change。

---

### T2.1 — 创建 devloop-intake Skill

**目标**: 阶段二编排 Skill。解析输入→查KB索引→影响分析→NFR评估🆕→生成需求文档→门禁→等确认→提交。

**复用**:
| 调用 | 用途 |
|------|------|
| `codegraph_explore` (MCP) | 按关键词查 KB 索引 |
| `brainstorming` | 影响分析（提供 KB 上下文作为输入） |
| *(编排层逻辑)* 🆕 | 非功能需求强制评估（加载 `templates/nfr-checklist.md`） |
| `openspec-propose` | 创建 OpenSpec Change（proposal.md + .comet.yaml） |
| `comet-state` (T0.2) | checkpoint |
| `devloop-guard` (T0.3) | 门禁 G2.1-G2.5（含 G2.3-NFR） |

**新建逻辑**（薄编排层）:
- 输入解析: 自然语言/用户故事/Bug报告 → 标准化结构（title, type, priority, description, acceptance_criteria）
- 索引式 KB 加载: 关键词提取 → `.manifest.yaml` 查询 → Top-N 匹配 → 摘要层降级（token 预算控制）
- 确认级别判定: 根据 type/priority/affected_modules 自动判定 Level 1/2/3（默认 Level 3）
- **非功能需求强制评估 🆕**: 影响分析完成后，加载 `templates/nfr-checklist.md`，遍历 7 个维度（认证授权/输入校验/数据保护/速率限制/审计日志/性能/可访问性），根据需求特征判定触发条件，对触发维度生成补充验收标准，追加到影响分析报告和需求文档中
- 需求文档生成: 使用 `templates/requirement-doc.md` 模板，LLM 填充变量（含 NFR 补充验收项）
- 基线需求用途限制: 如果用到了 baseline 需求作为背景参考，必须标注 `usage_restriction: reference_only`
- Git commit: `[devloop] stage:2 action:intake req:<id>`

**产出**: `.claude/skills/devloop-intake/SKILL.md`

**验证**:
```bash
test -s .claude/skills/devloop-intake/SKILL.md \
  && grep -q 'openspec-propose\|brainstorming' .claude/skills/devloop-intake/SKILL.md \
  && grep -q 'manifest.yaml\|关键词' .claude/skills/devloop-intake/SKILL.md \
  && grep -q 'nfr-checklist\|非功能需求' .claude/skills/devloop-intake/SKILL.md \
  && echo "✅ T2.1 PASSED" || echo "❌ T2.1 FAILED"
```

---

### T2.2 — 运行阶段二（首次需求摄入）

**目标**: 输入一个自然语言需求，验证完整阶段二流程。

**测试用例**: `"添加用户登录日志功能，记录每次登录的时间、IP 地址和 User-Agent 信息"`

**验证**:
```bash
# 检查需求文档
REQ_FILE=$(ls requirements/in-progress/REQ-*.md 2>/dev/null | head -1)
test -s "$REQ_FILE" && echo "✓ 需求文档: $REQ_FILE"
# 检查 frontmatter
grep -q 'status: proposed' "$REQ_FILE" && echo "✓ status: proposed"
grep -q 'kb_context:' "$REQ_FILE" && echo "✓ kb_context 已关联"
# 检查 OpenSpec Change
test -d "openspec/changes/"*/ && echo "✓ OpenSpec Change 已创建"
echo "✅ T2.2 PASSED"
```

---

### T2.3 — 确认流程 + 门禁校验

**目标**: 验证确认级别判定和门禁。

**实现**:
1. 检查确认级别是否正确判定（默认应为 Level 3）
2. 运行 `devloop-guard check stage-2`
3. 模拟人工确认流程: `confirmation-queue list` → `confirm`

**验证**:
```bash
.comet/scripts/devloop-guard check stage-2
# 检查确认项
.comet/scripts/comet-state get pending_confirmations
echo "✅ T2.3 PASSED"
```

---

## Phase 3: 阶段三 — 代码生成（2 个任务）

> **目标**: 根据需求文档，走 Design→Plan→Build 流程，产出代码变更。
> **完成标志**: 一个需求从 proposed 走到 built（代码已提交）。

---

### T3.1 — 创建 devloop-build Skill

**目标**: 阶段三编排 Skill。拉需求→委托设计→委托计划→委托执行→门禁→提交。

**复用**:
| 调用 | 用途 |
|------|------|
| `brainstorming` | 设计方案（提供需求文档+KB上下文作为输入） |
| `writing-plans` | 创建实现计划（生成 plan.md + tasks.md） |
| `subagent-driven-development` | 执行实现（大需求，>5 tasks，后台子代理） |
| `executing-plans` | 执行实现（小需求，<5 tasks，主会话） |
| `using-git-worktrees` | 隔离工作空间（可选，大需求推荐） |
| *(sdd 内部调用)* `test-driven-development` | 每个 task 的 TDD |
| *(sdd 内部调用)* `requesting-code-review` | 每个 task 的双审查 |
| `comet-state` (T0.2) | checkpoint |
| `devloop-guard` (T0.3) | 门禁 G3.1-G3.4 |

**新建逻辑**（薄编排层）:
- 需求拉取: 扫描 `requirements/in-progress/` → 优先级排序 → 选取最高分
- 确认点管理: 设计方案后暂停（Level 3/2），实现计划后暂停（Level 3）
- Build 模式选择: 根据 task 数量自动选 `executing-plans` 或 `subagent-driven-development`
- 产物路径映射: 将 design.md/plan.md/tasks.md 写入 `openspec/changes/<name>/`
- 进度追踪: 每个 task 完成后调用 `comet-state complete-step`
- Git commit: `[devloop] stage:3 action:build req:<id>`

**产出**: `.claude/skills/devloop-build/SKILL.md`

**验证**:
```bash
test -s .claude/skills/devloop-build/SKILL.md \
  && grep -q 'writing-plans\|subagent-driven-development\|brainstorming' \
     .claude/skills/devloop-build/SKILL.md \
  && echo "✅ T3.1 PASSED" || echo "❌ T3.1 FAILED"
```

---

### T3.2 — 运行阶段三（首个需求的 Design→Plan→Build）

**目标**: 使用 Phase 2 产出的需求文档，走完 Design→Plan→Build。

**前置**: T2.2 产生的需求（status: proposed）

**验证**:
```bash
# 检查产物
test -s openspec/changes/*/design.md && echo "✓ design.md"
test -s openspec/changes/*/plan.md && echo "✓ plan.md"
test -s openspec/changes/*/tasks.md && echo "✓ tasks.md"
# 检查代码变更
git log --oneline --grep="\[devloop\] stage:3" -3
# 检查门禁
.comet/scripts/devloop-guard check stage-3
echo "✅ T3.2 PASSED"
```

---

## Phase 4: 阶段四+五 — 验证与闭环（3 个任务）

> **目标**: 验证代码→审查→修复→提交→更新需求状态→触发下一个需求。
> **完成标志**: 一个需求完成闭环，自动触发第二个需求。

---

### T4.1 — 创建 devloop-verify Skill

**目标**: 阶段四编排 Skill。验证→审查→修复循环→提交→更新需求状态。

**复用**:
| 调用 | 用途 |
|------|------|
| `verification-before-completion` | 验证纪律（必须看到测试通过的实际输出） |
| `code-review` (内置) | 代码审查 |
| `systematic-debugging` | 修复失败（根因→假设→验证→修复） |
| `finishing-a-development-branch` | 收尾合并 |
| `comet-state` (T0.2) | checkpoint |
| `devloop-guard` (T0.3) | 门禁 G4.1-G4.4 |

**新建逻辑**（薄编排层）:
- 修复循环: 失败 ≤3 次自动重试，>3 次暂停等人工决策
- 验证策略: 聚焦输出正确性（测试通过=正确），不要求环境一致
- 需求状态更新: `status: completed` + `git mv in-progress/ → completed/`
- Git commit: `[devloop] stage:4 action:verify` + `[devloop] stage:5 action:complete`

**产出**: `.claude/skills/devloop-verify/SKILL.md`

**验证**:
```bash
test -s .claude/skills/devloop-verify/SKILL.md \
  && grep -q 'verification-before-completion\|systematic-debugging\|finishing-a-development-branch' \
     .claude/skills/devloop-verify/SKILL.md \
  && echo "✅ T4.1 PASSED" || echo "❌ T4.1 FAILED"
```

---

### T4.2 — 创建 devloop-loop Skill

**目标**: 阶段五编排 Skill。扫描队列→优先级排序→路由到阶段二或阶段三。

**复用**:
| 调用 | 用途 |
|------|------|
| `devloop-intake` (T2.1) | 路由: proposed 状态的需求 → 阶段二 |
| `devloop-build` (T3.1) | 路由: designed/planned 状态的需求 → 阶段三 |
| `comet-state` (T0.2) | 更新 active_requirement |
| `devloop-guard` (T0.3) | 门禁 G5.1 |

**新建逻辑**（薄编排层）:
- 队列扫描: `requirements/in-progress/` 中 status != completed/blocked
- 优先级排序: priority_score + dependency_score
- 状态路由:
  - `proposed` → 触发 `devloop-intake`（从需求完善开始）
  - `designed` → 触发 `devloop-build`（从 Plan 开始）
  - `planned` → 触发 `devloop-build`（从 Build 开始）
- 队列空 → 输出状态摘要，提示等待新需求

**产出**: `.claude/skills/devloop-loop/SKILL.md`

**验证**:
```bash
test -s .claude/skills/devloop-loop/SKILL.md \
  && grep -q '扫描\|路由\|priority' .claude/skills/devloop-loop/SKILL.md \
  && echo "✅ T4.2 PASSED" || echo "❌ T4.2 FAILED"
```

---

### T4.3 — 端到端验证 + 闭环测试

**目标**: 验证阶段四（验证→完成）+ 阶段五（自动触发下一需求）全链路。

**前置**: T3.2 产出代码变更；需要至少 2 个需求在队列中

**验证**:
```bash
# 1. 需求文档已移到 completed/
ls requirements/completed/REQ-*.md | head -3

# 2. 需求状态正确
grep 'status: completed' requirements/completed/REQ-*.md

# 3. Git log 完整
echo "=== Git 提交链 ==="
git log --oneline --grep="\[devloop\]" -10

# 4. 检查闭环：active_requirement 是否指向下一个需求
.comet/scripts/comet-state get active_requirement.id

echo "✅ T4.3 PASSED: 闭环流转正常"
```

---

## Phase 5: 防御与保鲜（3 个任务）

> **目标**: KB 漂移检测、定时保鲜、确认队列、信任评估全部就绪。
> **完成标志**: 漂移报告能正确识别过时条目，保鲜可手动触发。

---

### T5.1 — 漂移检测 + 信任评估

**目标**: 创建 `kb-drift-check` 和 `kb-trust-update` 脚本。

**实现**:
1. `.comet/scripts/kb-drift-check`: 对比 codegraph 当前符号 → KB 条目 → 生成 `.drift-report.yaml`
2. `.comet/scripts/kb-trust-update`: 扫描所有 KB 条目 frontmatter → 计算 trust level → 检测阻断条件

**验证**:
```bash
chmod +x .comet/scripts/kb-drift-check .comet/scripts/kb-trust-update
.comet/scripts/kb-drift-check
test -s knowledge-base/.drift-report.yaml && echo "✓ 漂移报告已生成"
.comet/scripts/kb-trust-update
echo "✅ T5.1 PASSED"
```

---

### T5.2 — 保鲜触发 + 确认队列

**目标**: 创建 `kb-freshness-trigger` 和 `confirmation-queue` 脚本。

**实现**:
1. `.comet/scripts/kb-freshness-trigger`: 支持 `--mode post-verify|cron|manual` + `--action report|fix`
2. `.comet/scripts/confirmation-queue`: 支持 `list|show|confirm|reject|skip|batch-confirm|check-timeout`

**验证**:
```bash
chmod +x .comet/scripts/kb-freshness-trigger .comet/scripts/confirmation-queue
.comet/scripts/kb-freshness-trigger --mode manual --scope all --action report
.comet/scripts/confirmation-queue list
echo "✅ T5.2 PASSED"
```

---

### T5.3 — 防御系统集成测试

**目标**: 验证漂移检测→信任评估→保鲜修复→阻断的全链路。

**验证**:
```bash
# 1. 全量漂移扫描
.comet/scripts/kb-drift-check
# 2. 信任级别评估
.comet/scripts/kb-trust-update
# 3. 模拟代码变更后保鲜
# （手动修改一个文件后:）
.comet/scripts/kb-freshness-trigger --mode manual --scope all --action report
# 4. 检查是否有阻断级漂移
cat knowledge-base/.drift-report.yaml
echo "✅ T5.3 PASSED"
```

---

## Phase 6: 系统验收（2 个任务）

> **目标**: 用 3 个不同类型需求验证端到端闭环 + 中断恢复。
> **完成标志**: 所有需求完成，中断恢复正常。

---

### T6.1 — 三需求端到端验收

**目标**: 依次处理 3 个不同类型/优先级的需求，验证确认分级和闭环流转。

**测试用例**:
| # | 类型 | 优先级 | 预期确认级别 | 预期流程 |
|---|------|--------|------------|---------|
| 1 | bugfix | low | Level 1 | 快速通道，零确认直到阶段四 |
| 2 | feature | high | Level 3 | 高保障，Design+Plan 暂停确认 |
| 3 | enhancement | medium | Level 2 | 标准保障，Design 确认，Plan 自动 |

**验证**:
```bash
COMPLETED=$(ls requirements/completed/REQ-*.md | wc -l)
echo "已完成需求数: $COMPLETED"
test "$COMPLETED" -ge 3 && echo "✅ T6.1 PASSED" || echo "❌ 需求数不足"
```

---

### T6.2 — 中断恢复测试

**目标**: 模拟会话中断，验证 `devloop-resume` Skill 正确恢复。

**测试场景**:
1. 启动需求处理流程
2. 在阶段三 Design 完成后（确认点前）中断
3. 运行 `comet-state check <name> build --recover`
4. 加载 `devloop-resume` Skill
5. 验证从 Design 确认点继续

**验证**:
```bash
# 模拟中断
.comet/scripts/comet-state init recovery-test
.comet/scripts/comet-state set current_stage 3
.comet/scripts/comet-state checkpoint 3 3.3 "design.md done, waiting confirm"
.comet/scripts/comet-state add-pending plan_ready REQ-TEST "等待确认设计方案"

# 运行恢复
OUTPUT=$(comet-state check recovery-test build --recover 2>&1)
echo "$OUTPUT" | grep -q "pending_confirmations\|resume_from_step" \
  && echo "✅ T6.2 PASSED" || echo "❌ T6.2 FAILED"
```

---

## 附录 A: 任务依赖关系

```
Phase 0                    Phase 1+2                 Phase 3+4              Phase 5+6
T0.0                       T1.1──→T1.2──→T1.3       T3.1──→T3.2            T5.1──→T5.2──→T5.3
  │                          │                          │                     │
  ├──→T0.1                   │       T2.1──→T2.2──→T2.3│                     │
  │     │                    │         │                │                     │
  ├──→T0.2──→T0.4           │         │                │                     │
  │     │                    │         │                │                     │
  └──→T0.3                  T1.3  ←── Phase 1 完成    T4.1──→T4.2──→T4.3    │
                              │         │              │                     │
                             T2.3  ←── Phase 2 完成  T4.3  ←── Phase 3+4    │
                                                        │                   │
                                                       T6.1──→T6.2  ←────────┘
```

## 附录 B: 最终产物全景

### 一次性基础设施（做好后不再改动，除非扩展）

```
项目根目录/
│
├── .claude/skills/              # Skill 定义
│   ├── devloop-reverse/SKILL.md   🆕 阶段一编排
│   ├── devloop-intake/SKILL.md    🆕 阶段二编排
│   ├── devloop-build/SKILL.md     🆕 阶段三编排
│   ├── devloop-verify/SKILL.md    🆕 阶段四编排
│   ├── devloop-loop/SKILL.md      🆕 阶段五编排
│   ├── devloop-resume/SKILL.md    🆕 中断恢复
│   ├── brainstorming/SKILL.md     ✅ 已有（设计/分析）
│   ├── writing-plans/SKILL.md     ✅ 已有（计划）
│   ├── subagent-driven-development/SKILL.md ✅ 已有（执行）
│   ├── executing-plans/SKILL.md   ✅ 已有（执行-小需求）
│   ├── test-driven-development/SKILL.md ✅ 已有（TDD）
│   ├── requesting-code-review/SKILL.md   ✅ 已有（审查）
│   ├── systematic-debugging/SKILL.md     ✅ 已有（调试）
│   ├── verification-before-completion/SKILL.md ✅ 已有（验证）
│   ├── finishing-a-development-branch/SKILL.md ✅ 已有（收尾）
│   ├── dispatching-parallel-agents/SKILL.md    ✅ 已有（并行）
│   └── using-git-worktrees/SKILL.md      ✅ 已有（隔离）
│
├── .comet/
│   ├── scripts/
│   │   ├── comet-state            🆕 状态管理
│   │   ├── devloop-guard          🆕 门禁引擎
│   │   ├── kb-drift-check         🆕 漂移检测
│   │   ├── kb-freshness-trigger   🆕 保鲜触发
│   │   ├── kb-trust-update        🆕 信任评估
│   │   └── confirmation-queue     🆕 确认队列
│   ├── guard-config.yaml          🆕 18个门禁定义（硬/软分离）
│   ├── gate-results/               🆕 软门禁 LLM 评估结果暂存
│   └── loop-state.yaml            🆕 运行时状态（初始为空）
│
├── templates/                     🆕 10个产出物模板
│   ├── requirement-input.md
│   ├── requirement-doc.md         # 含基线模式 AUTO-INFERRED 警告块
│   ├── kb-module-overview.md
│   ├── kb-module-api.md
│   ├── kb-module-data-model.md
│   ├── kb-module-business-logic.md
│   ├── kb-module-dependencies.md
│   ├── impact-analysis.md
│   ├── nfr-checklist.md           🆕 非功能需求强制评估清单（7维度）
│   └── verification-checklist.md
│
└── design/                        ✅ 已有（本次设计文档）
```

### 运行时产物（每次运行 DevLoop 产生）

```
knowledge-base/
├── .manifest.yaml              # 模块索引（关键词/实体/API/token估算）
├── .drift-report.yaml          # 漂移报告（含 soft_gate_warnings 字段）
├── architecture/
│   ├── overview.md             # 整体架构
│   ├── components.md           # 组件拓扑
│   └── data-flow.md            # 数据流
├── modules/<name>/             # 每个模块5个KB文件
│   ├── overview.md
│   ├── api.md
│   ├── data-model.md
│   ├── business-logic.md
│   └── dependencies.md
├── apis/internal.md
└── data-models/global.md

requirements/
├── baseline/REQ-001-*.md       # 基线需求（从代码反推）
├── in-progress/REQ-010-*.md    # 进行中的需求
└── completed/REQ-005-*.md      # 已完成的需求

openspec/changes/<name>/        # 每个需求的变更目录
├── .comet.yaml
├── proposal.md                 # 需求提案
├── design.md                   # 设计方案（brainstorming产出）
├── plan.md                     # 实现计划（writing-plans产出）
├── tasks.md                    # 任务清单（writing-plans产出）
└── specs/<name>/spec.md        # Delta spec

.comet/loop-state.yaml          # 当前进度状态
```

## 附录 C: 复用 Skill 调用关系图

```
devloop-reverse
  ├──→ codegraph_explore (MCP)        扫描代码，生成模块清单
  ├──→ dispatching-parallel-agents    并行分析各模块
  │     └── 每个 agent 内部:
  │         └──→ codegraph_node (MCP)   读取模块符号/API
  ├──→ brainstorming                  KB汇总→反推需求
  ├──→ comet-state                    checkpoint
  └──→ devloop-guard                  门禁G1

devloop-intake
  ├──→ (内部) 输入解析                标准化需求结构
  ├──→ codegraph_explore (MCP)        查询KB索引
  ├──→ brainstorming                  影响分析
  ├──→ openspec-propose               创建OpenSpec Change
  ├──→ comet-state                    checkpoint
  ├──→ devloop-guard                  门禁G2
  └──→ confirmation-queue             等确认

devloop-build
  ├──→ (内部) 拉取+排序需求
  ├──→ brainstorming                  设计方案
  ├──→ writing-plans                  实现计划
  ├──→ subagent-driven-development    执行（大需求）
  │     └── 内部自动调用:
  │         ├──→ test-driven-development   每个task的TDD
  │         └──→ requesting-code-review    每个task的双审查
  │     或
  ├──→ executing-plans                执行（小需求）
  ├──→ using-git-worktrees            隔离（可选）
  ├──→ comet-state                    checkpoint
  └──→ devloop-guard                  门禁G3

devloop-verify
  ├──→ verification-before-completion 验证纪律
  ├──→ code-review (内置)             代码审查
  ├──→ systematic-debugging           失败修复
  ├──→ finishing-a-development-branch 收尾合并
  ├──→ comet-state                    checkpoint
  └──→ devloop-guard                  门禁G4

devloop-loop
  ├──→ (内部) 扫描+排序
  ├──→ devloop-intake 或 devloop-build  路由
  └──→ devloop-guard                  门禁G5
```
