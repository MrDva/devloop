# 05 — DevLoop 实现任务清单

> **目标**: 按照设计文档逐步实现 DevLoop 闭环开发系统，每个任务独立可验证。
> **使用方式**: 每个任务可以直接交给 Claude Code 执行，按编号顺序进行。
> **验证原则**: 每个任务完成后运行"验证"命令，确认通过后再进入下一个任务。

---

## 总览

| Phase | 名称 | 任务数 | 预计时间 | 关键产出 |
|-------|------|--------|---------|---------|
| Phase 0 | 基础建设 | 7 | 2-3h | 目录结构、模板、state/guard 脚本 |
| Phase 1 | 阶段一：逆向分析 | 6 | 3-4h | devloop-reverse Skill、首个 KB 产物 |
| Phase 2 | 阶段二：需求摄入 | 5 | 2-3h | devloop-intake Skill、结构化需求文档 |
| Phase 3 | 阶段三：代码生成 | 4 | 2-3h | devloop-build Skill、代码变更产物 |
| Phase 4 | 阶段四+五：验证与闭环 | 5 | 2-3h | devloop-verify/loop Skill、闭环验证 |
| Phase 5 | 防御与保鲜 | 5 | 2-3h | 漂移检测、定时保鲜、确认队列 |
| Phase 6 | 端到端集成测试 | 4 | 2-3h | 完整闭环跑通 3 个需求 |
| **合计** | | **36** | **14-22h** | |

---

## Phase 0: 基础建设

> **目标**: 搭建 DevLoop 运行所需的目录结构、模板文件、状态管理和门禁执行基础脚本。
> **完成标志**: `comet-state` 和 `devloop-guard` 脚本可通过命令行独立运行。

---

### T0.1 — 创建 DevLoop 目录结构

**目标**: 一键创建所有必要的目录，为后续任务提供骨架。

**输入**: 无（首个任务）

**实现步骤**:
1. 创建 `knowledge-base/` 目录，含子目录: `architecture/`, `modules/`, `apis/`, `data-models/`
2. 创建 `requirements/` 目录，含子目录: `baseline/`, `in-progress/`, `completed/`
3. 创建 `templates/` 目录
4. 创建 `.comet/` 子目录 (如果不存在)
5. 在项目根目录 `.gitignore` 中确认或追加: `.comet/loop-state.yaml` 不忽略（需要 Git 追踪）

**产出**:
```
knowledge-base/
├── architecture/
├── modules/
├── apis/
└── data-models/
requirements/
├── baseline/
├── in-progress/
└── completed/
templates/
```

**验证**:
```bash
test -d knowledge-base/architecture && \
test -d knowledge-base/modules && \
test -d knowledge-base/apis && \
test -d knowledge-base/data-models && \
test -d requirements/baseline && \
test -d requirements/in-progress && \
test -d requirements/completed && \
test -d templates && \
echo "✅ T0.1 PASSED: All directories created"
```

---

### T0.2 — 创建模板文件

**目标**: 创建所有产出物的 Markdown 模板，包含 frontmatter 和 `{{ }}` 占位符。

**输入**: T0.1（目录结构存在）

**实现步骤**:
1. 创建 `templates/requirement-input.md` — 外部需求输入模板（用户提交需求时填写）
2. 创建 `templates/requirement-doc.md` — 统一需求文档模板（阶段二产出）
3. 创建 `templates/kb-module-overview.md` — KB 模块概述模板（阶段一产出）
4. 创建 `templates/kb-module-api.md` — KB API 文档模板
5. 创建 `templates/kb-module-data-model.md` — KB 数据模型模板
6. 创建 `templates/kb-module-business-logic.md` — KB 业务逻辑模板
7. 创建 `templates/kb-module-dependencies.md` — KB 依赖关系模板
8. 创建 `templates/impact-analysis.md` — 影响分析报告模板
9. 创建 `templates/verification-checklist.md` — 验证检查清单模板

**模板规范**: 每个模板必须包含:
- YAML frontmatter（`---` 包裹），包含必填字段
- `{{ variable }}` 占位符用于运行时填充
- 所有占位符在模板末尾以注释形式列出（作为变量清单）

**产出**:
```
templates/
├── requirement-input.md
├── requirement-doc.md
├── kb-module-overview.md
├── kb-module-api.md
├── kb-module-data-model.md
├── kb-module-business-logic.md
├── kb-module-dependencies.md
├── impact-analysis.md
└── verification-checklist.md
```

**验证**:
```bash
# 每个模板文件需要存在且非空，包含 frontmatter 和至少一个 {{ }} 占位符
for f in requirement-input requirement-doc kb-module-overview kb-module-api \
         kb-module-data-model kb-module-business-logic kb-module-dependencies \
         impact-analysis verification-checklist; do
  t="templates/${f}.md"
  test -s "$t" || { echo "FAIL: $t is empty or missing"; exit 1; }
  grep -q '^---$' "$t" || { echo "FAIL: $t missing frontmatter"; exit 1; }
  grep -q '{{.*}}' "$t" || { echo "FAIL: $t missing variable placeholders"; exit 1; }
  echo "  ✓ $t"
done
echo "✅ T0.2 PASSED: All templates created with frontmatter and variables"
```

---

### T0.3 — 创建 KB 清单模板（`.manifest.yaml`）

**目标**: 创建知识库索引文件模板，支持模块级关键词、token 估算和依赖关系。

**输入**: T0.1（目录结构存在）

**实现步骤**:
1. 创建 `templates/kb-manifest.yaml` — KB 清单模板
2. 模板包含: 模块名称、路径、关键词列表、实体列表、API 列表、token 估算、依赖关系
3. 同时创建 `knowledge-base/.manifest.yaml` 作为初始空文件（包含空 `modules: []`）

**产出**:
```
templates/kb-manifest.yaml
knowledge-base/.manifest.yaml
```

**验证**:
```bash
test -s templates/kb-manifest.yaml && \
test -f knowledge-base/.manifest.yaml && \
grep -q 'modules:' knowledge-base/.manifest.yaml && \
echo "✅ T0.3 PASSED: Manifest template and initial file created"
```

---

### T0.4 — 实现状态管理脚本（`comet-state`）

**目标**: 实现 `comet-state` Bash 脚本，支持 `set`/`get`/`check --recover` 操作。

**输入**: T0.1（目录结构存在）

**实现步骤**:
1. 创建 `.comet/scripts/comet-state` Bash 脚本
2. 实现子命令:
   - `comet-state init <change-name>` — 初始化 loop-state.yaml
   - `comet-state set <key> <value>` — 设置状态字段
   - `comet-state get <key>` — 读取状态字段
   - `comet-state checkpoint <stage> <step> "<description>"` — 保存 checkpoint
   - `comet-state complete-step <stage> <step>` — 标记步骤完成
   - `comet-state check <change-name> <phase> --recover` — 恢复检查
   - `comet-state gate-result <gate-id> <passed|failed> [message]` — 记录门禁结果
   - `comet-state add-pending-confirmation <type> <req> "<message>"` — 添加确认
   - `comet-state scale <change-name>` — 显示当前状态摘要
3. 脚本以 YAML 格式读写 `.comet/loop-state.yaml`
4. 使用 `yq` 操作 YAML（如不可用则输出提示安装）
5. 添加错误处理: 文件不存在时提示先 `init`

**产出**:
```
.comet/scripts/comet-state        (可执行脚本)
.comet/loop-state.yaml            (首次运行 init 时创建)
```

**验证**:
```bash
chmod +x .comet/scripts/comet-state

# 测试完整生命周期
.comet/scripts/comet-state init test-change
.comet/scripts/comet-state set current_stage 1
.comet/scripts/comet-state set active_requirement.id REQ-TEST
.comet/scripts/comet-state checkpoint 1 1.2 "module analysis done"
.comet/scripts/comet-state complete-step 1 1.1
.comet/scripts/comet-state gate-result G1.1 passed "coverage 85%"
.comet/scripts/comet-state get current_stage

# 验证输出
STAGE=$(.comet/scripts/comet-state get current_stage)
test "$STAGE" = "1" && echo "✅ T0.4 PASSED: comet-state works correctly" || echo "❌ T0.4 FAILED"
```

---

### T0.5 — 实现门禁执行脚本（`devloop-guard`）

**目标**: 实现门禁执行引擎，支持 `block_and_retry`/`warn_and_continue`/`reject`/`pause_for_human` 四种策略。

**输入**: T0.1（目录结构存在）

**实现步骤**:
1. 创建 `.comet/scripts/devloop-guard` Bash 脚本
2. 创建 `templates/guard-config.yaml` — 门禁配置模板
3. 创建 `.comet/guard-config.yaml` — 项目级门禁配置（初始使用默认值）
4. 实现子命令:
   - `devloop-guard check <stage>` — 运行指定阶段的所有门禁
   - `devloop-guard check <gate-id>` — 运行单个门禁
   - `devloop-guard list` — 列出所有门禁及其状态
   - `devloop-guard override <gate-id> "<reason>"` — 人工覆盖门禁
5. 门禁规则使用简单表达式:
   - 文件存在性检查: `file_exists:<path>`
   - 计数检查: `count:<path> >= <N>`
   - Git 状态检查: `git_clean`
6. 输出格式: 每个门禁一行，`[PASS]`/`[WARN]`/`[FAIL]` + 详情
7. 最终输出必须包含 `ALL CHECKS PASSED` 或列出失败项

**产出**:
```
.comet/scripts/devloop-guard        (可执行脚本)
.comet/guard-config.yaml            (门禁配置)
templates/guard-config.yaml         (模板)
```

**验证**:
```bash
chmod +x .comet/scripts/devloop-guard

# 列出门禁
.comet/scripts/devloop-guard list

# 运行一个简单门禁（文件存在性检查应通过）
echo "knowledge-base/.manifest.yaml" > /tmp/test_guard
# 检查 guard 脚本是否可以解析 guard-config.yaml 并输出预期格式
.comet/scripts/devloop-guard list 2>&1 | grep -q "G1.1\|G1.2\|G2.1" && \
echo "✅ T0.5 PASSED: devloop-guard lists gates" || echo "❌ T0.5 FAILED"
```

---

### T0.6 — 创建 devloop-state 恢复 Skill

**目标**: 创建 Skill，使 Claude Code 可以在会话恢复时自动读取状态并继续未完成的流程。

**输入**: T0.4（comet-state 脚本可用）

**实现步骤**:
1. 创建 `.claude/skills/devloop-resume/SKILL.md`
2. Skill 内容: 读取 `.comet/loop-state.yaml` → 展示当前进度 → 处理待确认项 → 从 checkpoint 继续
3. Skill 调用 `comet-state check --recover` 确定恢复动作
4. 包含上下文快速重建逻辑（读取 proposal/KB/memory）

**产出**:
```
.claude/skills/devloop-resume/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/devloop-resume/SKILL.md && \
grep -q 'name:' .claude/skills/devloop-resume/SKILL.md && \
grep -q 'comet-state' .claude/skills/devloop-resume/SKILL.md && \
echo "✅ T0.6 PASSED: devloop-resume skill created"
```

---

### T0.7 — 创建全局 guard-config.yaml

**目标**: 将设计文档中的所有门禁（G1.1-G5.1）转化为可执行的 YAML 配置文件。

**输入**: T0.5（devloop-guard 脚本可用）

**实现步骤**:
1. 根据 `02-detailed-design.md` 第 7.3 节的完整门禁矩阵，填充 `.comet/guard-config.yaml`
2. 每个门禁条目包含: `id`, `name`, `description`, `stage`, `rule`, `on_fail`, `max_retries`
3. 规则使用 devloop-guard 支持的表达式语法
4. 添加 Phase 0 的初始门禁（这些门禁在基础设施就绪后应立即通过）

**门禁清单**（来自设计文档）:

| ID | 名称 | 阶段 | 类型 | on_fail |
|----|------|------|------|---------|
| G1.1 | 模块覆盖率 | 1 | 统计 | warn_and_continue |
| G1.2 | KB 条目完整性 | 1 | 结构 | block_and_retry |
| G1.3 | KB 内部一致性 | 1 | 引用 | block_and_retry |
| G1.4 | 需求反向覆盖率 | 1 | 统计 | warn_and_continue |
| G1.5 | 阶段一总门禁 | 1 | 汇总 | block_and_retry |
| G2.1 | 输入有效性 | 2 | 格式 | reject |
| G2.2 | KB 上下文相关性 | 2 | 语义 | warn_and_continue |
| G2.3 | 影响分析完整性 | 2 | 结构 | block_and_retry |
| G2.4 | 需求文档结构 | 2 | 格式 | block_and_retry |
| G2.5 | 阶段二总门禁 | 2 | 汇总 | block_and_retry |
| G3.1 | 需求可构建性 | 3 | 前置 | block_and_retry |
| G3.2 | Design 阶段质量 | 3 | 过程 | block_and_retry |
| G3.3 | Plan 阶段质量 | 3 | 过程 | block_and_retry |
| G3.4 | Build 阶段质量 | 3 | 过程 | block_and_retry |
| G4.1 | 自动化验证 | 4 | 测试 | block_and_retry |
| G4.2 | 代码审查 | 4 | 审查 | block_and_retry |
| G4.3 | 修复循环限制 | 4 | 过程 | pause_for_human |
| G4.4 | 阶段四总门禁 | 4 | 汇总 | block_and_retry |
| G5.1 | 闭环状态一致性 | 5 | 状态 | block_and_retry |

**产出**:
```
.comet/guard-config.yaml    (18 个门禁的完整配置)
```

**验证**:
```bash
# 验证 YAML 结构完整性
grep -c "id:" .comet/guard-config.yaml | xargs -I{} test {} -ge 18 && \
.comet/scripts/devloop-guard list 2>&1 | grep -c "G" | xargs -I{} test {} -ge 18 && \
echo "✅ T0.7 PASSED: All 18 gates configured"
```

---

## Phase 1: 阶段一 — 逆向分析

> **目标**: 实现 `devloop-reverse` Skill，能从现有代码生成结构化 KB 和基线需求文档。
> **完成标志**: 在当前项目上运行阶段一，成功产出一个模块的完整 KB 条目。

---

### T1.1 — 创建 KB 模块条目生成 Skill

**目标**: 创建 `kb-module-extract` Skill，对单个模块进行深度分析并生成结构化 KB 条目。

**输入**: Phase 0 完成；T0.2（模板可用）

**实现步骤**:
1. 创建 `.claude/skills/kb-module-extract/SKILL.md`
2. Skill 输入: 模块名、模块路径
3. 利用 CodeGraph MCP (`codegraph_explore`) 获取模块的符号、API、依赖信息
4. 根据模板生成:
   - `knowledge-base/modules/<name>/overview.md`
   - `knowledge-base/modules/<name>/api.md`
   - `knowledge-base/modules/<name>/data-model.md`
   - `knowledge-base/modules/<name>/business-logic.md`
   - `knowledge-base/modules/<name>/dependencies.md`
5. 每个文件 frontmatter 包含防御标记: `source: auto-generated`, `confidence: <0-1>`, `drift_score: 0`
6. 单模块上下文控制在 < 50K tokens（超大模块需拆分子模块）

**产出**:
```
.claude/skills/kb-module-extract/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/kb-module-extract/SKILL.md && \
grep -q 'codegraph_explore\|codegraph_node' .claude/skills/kb-module-extract/SKILL.md && \
grep -q 'knowledge-base/modules/' .claude/skills/kb-module-extract/SKILL.md && \
echo "✅ T1.1 PASSED: kb-module-extract skill created"
```

---

### T1.2 — 创建代码扫描 Skill（生成模块清单）

**目标**: 利用 CodeGraph 扫描整个项目，生成 `knowledge-base/.manifest.yaml`。

**输入**: T1.1 (kb-module-extract Skill)，T0.3 (manifest 模板)

**实现步骤**:
1. 创建 `.claude/skills/kb-code-scan/SKILL.md`
2. Skill 逻辑:
   - 通过 CodeGraph 或目录扫描识别顶层模块
   - 按细粒度拆分策略（见设计 14.2 节）划分模块边界
   - 生成 `knowledge-base/.manifest.yaml`，包含每个模块的: name, path, type, files, dependencies, keywords, entities, apis, total_tokens_estimate
3. 门禁 G1.1: 模块覆盖率 >= 80%

**产出**:
```
.claude/skills/kb-code-scan/SKILL.md
knowledge-base/.manifest.yaml  (填充了实际模块列表)
```

**验证**:
```bash
# 检查 manifest 包含模块列表
test -s knowledge-base/.manifest.yaml && \
grep -q 'name:' knowledge-base/.manifest.yaml && \
grep -q 'path:' knowledge-base/.manifest.yaml && \
echo "✅ T1.2 PASSED: code scan skill created, manifest populated"
```

---

### T1.3 — 对单个模块运行 kb-module-extract（首次验证）

**目标**: 对项目中的一个真实模块运行 KB 提取，验证产出质量。

**输入**: T1.1 + T1.2 完成

**实现步骤**:
1. 从 `knowledge-base/.manifest.yaml` 中选择一个模块
2. 使用 Skill 工具加载 `kb-module-extract`
3. 对该模块生成完整的 5 个 KB 文件
4. 验证每个文件 frontmatter 包含防御标记
5. 运行门禁 G1.2

**产出**:
```
knowledge-base/modules/<module-name>/
├── overview.md
├── api.md
├── data-model.md
├── business-logic.md
└── dependencies.md
```

**验证**:
```bash
MODULE_NAME="<实际模块名>"
for f in overview api data-model business-logic dependencies; do
  test -s "knowledge-base/modules/${MODULE_NAME}/${f}.md" || \
    { echo "FAIL: ${f}.md missing"; exit 1; }
done
# 验证 frontmatter 防御标记
grep -q 'source: auto-generated' "knowledge-base/modules/${MODULE_NAME}/overview.md" && \
grep -q 'confidence:' "knowledge-base/modules/${MODULE_NAME}/overview.md" && \
echo "✅ T1.3 PASSED: Module KB entries generated with defense markers"
```

---

### T1.4 — 实现 KB 汇总整合（architecture + apis + data-models）

**目标**: 从各模块 KB 条目汇总生成项目级 KB 文档。

**输入**: T1.3（至少有一个模块的 KB 条目）

**实现步骤**:
1. 创建 `.claude/skills/kb-aggregate/SKILL.md`
2. Skill 逻辑:
   - 汇总所有模块的 overview → `knowledge-base/architecture/overview.md`
   - 汇总模块拓扑 → `knowledge-base/architecture/components.md`
   - 汇总数据流 → `knowledge-base/architecture/data-flow.md`
   - 汇总所有 API → `knowledge-base/apis/internal.md`
   - 汇总全局数据模型 → `knowledge-base/data-models/global.md`
3. 所有交叉引用使用 `[[module-name]]` 语法
4. 门禁 G1.3: 内部引用一致性

**产出**:
```
.claude/skills/kb-aggregate/SKILL.md
knowledge-base/architecture/overview.md
knowledge-base/architecture/components.md
knowledge-base/architecture/data-flow.md
knowledge-base/apis/internal.md
knowledge-base/data-models/global.md
```

**验证**:
```bash
test -s .claude/skills/kb-aggregate/SKILL.md && \
test -s knowledge-base/architecture/overview.md && \
test -s knowledge-base/architecture/components.md && \
echo "✅ T1.4 PASSED: KB aggregation complete"
```

---

### T1.5 — 实现需求反向推断

**目标**: 从完整 KB 反推基线需求文档，写入 `requirements/baseline/`。

**输入**: T1.4（完整 KB 可用）

**实现步骤**:
1. 创建 `.claude/skills/req-reverse-infer/SKILL.md`
2. Skill 逻辑:
   - 遍历所有已分析模块
   - 对每个模块: 读取 KB 条目 → brainstorming 推断需求 → 生成需求文档
   - 每个需求文档包含 frontmatter: `id`, `title`, `module`, `status: baseline`, `inferred_from`, `confidence`
   - 写入 `requirements/baseline/REQ-<NNN>-<slug>.md`
3. 门禁 G1.4: 需求覆盖率 >= 90%；confidence: low 不超过 20%

**产出**:
```
.claude/skills/req-reverse-infer/SKILL.md
requirements/baseline/REQ-001-*.md ... (每个模块至少一个)
```

**验证**:
```bash
test -s .claude/skills/req-reverse-infer/SKILL.md && \
BASELINE_COUNT=$(ls requirements/baseline/REQ-*.md 2>/dev/null | wc -l) && \
test "$BASELINE_COUNT" -gt 0 && \
echo "✅ T1.5 PASSED: $BASELINE_COUNT baseline requirements generated"
```

---

### T1.6 — 创建 devloop-reverse 总编排 Skill + 端到端测试

**目标**: 创建阶段一的总编排 Skill，串联 T1.2→T1.3→T1.4→T1.5→门禁→Git commit。

**输入**: T1.2-T1.5 全部完成

**实现步骤**:
1. 创建 `.claude/skills/devloop-reverse/SKILL.md`
2. Skill 按顺序调用子 Skill:
   - `kb-code-scan` → G1.1
   - 对每个模块并行调用 `kb-module-extract` → G1.2
   - `kb-aggregate` → G1.3
   - `req-reverse-infer` → G1.4
   - 总门禁 G1.5
   - Git commit（`[devloop] stage:1`）
3. 支持增量模式: 检测已有 KB，只分析变更模块
4. Skill 需要调用 `comet-state` 保存每一步 checkpoint
5. 完成后运行端到端测试: 在当前项目上执行完整阶段一

**产出**:
```
.claude/skills/devloop-reverse/SKILL.md
Git commit: [devloop] stage:1 action:reverse
```

**端到端验证**:
```bash
# 1. 清理旧 KB（如果存在）
rm -rf knowledge-base/modules/* knowledge-base/architecture/* knowledge-base/apis/* knowledge-base/data-models/* requirements/baseline/*

# 2. 加载 devloop-reverse Skill，对当前项目执行阶段一
#    （在 Claude Code 对话中执行: /devloop-reverse）

# 3. 验证产物
echo "=== KB 模块数 ===" && ls knowledge-base/modules/ | wc -l
echo "=== 基线需求数 ===" && ls requirements/baseline/ | wc -l
echo "=== Git log ===" && git log --oneline --grep="\[devloop\] stage:1" -1

# 4. 运行门禁
.comet/scripts/devloop-guard check stage-1

# 检查输出是否包含 "ALL CHECKS PASSED" 或列出可接受的警告数
echo "✅ T1.6 PASSED: Phase 1 end-to-end test complete"
```

---

## Phase 2: 阶段二 — 需求摄入

> **目标**: 实现 `devloop-intake` Skill，能接收外部需求输入，结合 KB 生成结构化需求文档。
> **完成标志**: 输入一个自然语言需求，成功生成 `requirements/in-progress/` 下的需求文档。

---

### T2.1 — 创建需求输入解析 Skill

**目标**: 解析多种格式的需求输入，标准化为内部数据结构。

**输入**: Phase 1 完成；T0.2（requirement-input.md 模板）

**实现步骤**:
1. 创建 `.claude/skills/req-parse-input/SKILL.md`
2. 支持输入格式: 自然语言、用户故事模板、Bug 报告、技术规格
3. 输出标准化结构:
   ```yaml
   title: "..."
   type: feature|bugfix|enhancement|refactor
   priority: high|medium|low
   source: cli|api|ide|webhook
   description: "..."
   related_modules: []
   acceptance_criteria: []
   ```
4. 门禁 G2.1: 输入有效性（至少 title + description）

**产出**:
```
.claude/skills/req-parse-input/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/req-parse-input/SKILL.md && \
grep -q 'title' .claude/skills/req-parse-input/SKILL.md && \
grep -q 'type.*feature\|bugfix' .claude/skills/req-parse-input/SKILL.md && \
echo "✅ T2.1 PASSED: req-parse-input skill created"
```

---

### T2.2 — 创建索引式 KB 加载 Skill

**目标**: 根据需求关键词，从 KB 中按需加载相关条目，而非全量加载。

**输入**: Phase 1 完成（KB 和 manifest 可用）

**实现步骤**:
1. 创建 `.claude/skills/kb-indexed-load/SKILL.md`
2. Skill 逻辑:
   - 从需求输入提取关键词（模块名、函数名、数据实体）
   - 查询 `knowledge-base/.manifest.yaml` 中的 keywords/entities/apis 字段
   - 按相关度排序，取 Top-N（N 由上下文预算决定，默认 3）
   - 优先加载 overview.md（摘要层），token 不足时降级
   - 输出: 相关 KB 条目列表 + 全局 architecture 摘要
3. 门禁 G2.2: 至少匹配 1 个相关模块，否则标记 confidence: low

**产出**:
```
.claude/skills/kb-indexed-load/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/kb-indexed-load/SKILL.md && \
grep -q 'manifest.yaml\|keywords\|相关' .claude/skills/kb-indexed-load/SKILL.md && \
echo "✅ T2.2 PASSED: kb-indexed-load skill created"
```

---

### T2.3 — 创建影响分析 Skill

**目标**: 基于需求 + KB 上下文，生成影响分析报告。

**输入**: T2.1 + T2.2

**实现步骤**:
1. 创建 `.claude/skills/req-impact-analysis/SKILL.md`
2. Skill 逻辑:
   - 加载需求 + KB 相关条目
   - 使用 `brainstorming` 分析影响范围
   - 生成影响分析报告（使用 `templates/impact-analysis.md`）
   - 报告必须包含: 涉及模块、数据模型变更、API 变更、风险评估
3. 门禁 G2.3: 必须包含三个必须部分；涉及 >3 模块时必须有风险评估

**产出**:
```
.claude/skills/req-impact-analysis/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/req-impact-analysis/SKILL.md && \
grep -q 'brainstorming' .claude/skills/req-impact-analysis/SKILL.md && \
grep -q '涉及模块\|影响' .claude/skills/req-impact-analysis/SKILL.md && \
echo "✅ T2.3 PASSED: req-impact-analysis skill created"
```

---

### T2.4 — 创建需求文档生成 Skill

**目标**: 综合输入解析、KB 上下文、影响分析，生成 OpenSpec Change + 统一需求文档。

**输入**: T2.1 + T2.2 + T2.3

**实现步骤**:
1. 创建 `.claude/skills/req-generate-doc/SKILL.md`
2. Skill 逻辑:
   - 调用 `comet-open` 或 `openspec-propose` 创建 OpenSpec Change
   - 使用 `templates/requirement-doc.md` 生成需求文档
   - 填入所有变量: 元数据、影响分析结果、验收标准、KB 上下文引用
   - 分配需求 ID（按序递增）
   - 写入 `requirements/in-progress/REQ-<NNN>-<slug>.md`
   - 更新需求文档 frontmatter: `status: proposed`
   - 在 `openspec/changes/<change-name>/` 下创建 proposal.md
3. 门禁 G2.4: frontmatter 完整、kb_context 引用 >= 2、与 proposal.md 一致

**产出**:
```
.claude/skills/req-generate-doc/SKILL.md
requirements/in-progress/REQ-*.md  (首次运行生成)
openspec/changes/<name>/proposal.md
```

**验证**:
```bash
test -s .claude/skills/req-generate-doc/SKILL.md && \
grep -q 'comet-open\|openspec-propose' .claude/skills/req-generate-doc/SKILL.md && \
grep -q 'requirement-doc' .claude/skills/req-generate-doc/SKILL.md && \
echo "✅ T2.4 PASSED: req-generate-doc skill created"
```

---

### T2.5 — 创建 devloop-intake 总编排 Skill + 端到端测试

**目标**: 创建阶段二的总编排 Skill，串联 T2.1→T2.2→T2.3→T2.4→门禁→Git commit→人工确认。

**输入**: T2.1-T2.4 全部完成

**实现步骤**:
1. 创建 `.claude/skills/devloop-intake/SKILL.md`
2. Skill 按顺序调用:
   - `req-parse-input` → G2.1
   - `kb-indexed-load` → G2.2
   - `req-impact-analysis` → G2.3
   - `req-generate-doc` → G2.4
   - 总门禁 G2.5
   - **暂停**: 展示需求摘要，等待用户确认（默认 Level 3）
   - Git commit（`[devloop] stage:2 action:intake req:<id>`）
3. 实现确认级别判定逻辑（根据需求 type/priority 自动判定 Level 3/2/1）
4. 完成后端到端测试: 输入一个自然语言需求，走完完整阶段二

**产出**:
```
.claude/skills/devloop-intake/SKILL.md
Git commit: [devloop] stage:2 action:intake
```

**端到端验证**:
```bash
# 在 Claude Code 对话中执行:
# "请对以下需求执行阶段二摄入: '添加用户登录日志功能，记录每次登录的时间、IP、User-Agent'"

# 验证
echo "=== 需求文档 ===" && ls -la requirements/in-progress/REQ-*.md
echo "=== OpenSpec Change ===" && ls openspec/changes/ | head -5
echo "=== 需求文档 frontmatter ===" && head -30 requirements/in-progress/REQ-*.md
# 检查 status: proposed 和 kb_context 引用
grep -q 'status: proposed' requirements/in-progress/REQ-*.md && \
echo "✅ T2.5 PASSED: Phase 2 end-to-end test complete"
```

---

## Phase 3: 阶段三 — 代码生成

> **目标**: 利用 Comet 体系实现 Design → Plan → Build 流程，编排已有 Skill。
> **完成标志**: 根据一个需求文档，走完 Design→Plan→Build，产出通过门禁的代码变更。

---

### T3.1 — 创建需求拉取与选取 Skill

**目标**: 从 Git 扫描待处理需求，按优先级排序选取下一个待构建需求。

**输入**: Phase 2 完成（有需求在 in-progress/）

**实现步骤**:
1. 创建 `.claude/skills/req-select-next/SKILL.md`
2. Skill 逻辑:
   - 扫描 `requirements/in-progress/` 中 status != completed/blocked 的需求
   - 按 priority_score（priority_weight + dependency_weight）排序
   - 选取最高分需求
   - 验证 G3.1: status 为 proposed/designed/planned，kb_context 引用有效
   - 更新 `loop-state.yaml` 中的 `active_requirement`

**产出**:
```
.claude/skills/req-select-next/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/req-select-next/SKILL.md && \
grep -q 'priority\|选取\|选取' .claude/skills/req-select-next/SKILL.md && \
echo "✅ T3.1 PASSED: req-select-next skill created"
```

---

### T3.2 — 创建 devloop-build 总编排 Skill（Design + Plan 部分）

**目标**: 编排 Comet Design 和 Comet Plan 阶段，调用已有 Comet Skill。

**输入**: T3.1；已有 Skill: `comet-design`, `writing-plans`

**实现步骤**:
1. 创建 `.claude/skills/devloop-build/SKILL.md`
2. Design 部分:
   - 加载需求文档 + KB 上下文
   - 调用 `comet-design`（已有 Skill）
   - 生成 design.md + delta spec
   - 门禁 G3.2
   - **暂停**: 展示设计方案，等待用户确认（Level 3/2）
3. Plan 部分:
   - 调用 `writing-plans`（已有 Skill）
   - 生成 plan.md + tasks.md（任务拆分，每个 task < 200 行变更）
   - 门禁 G3.3
   - **暂停**: 展示实现计划，等待用户确认（Level 3）
4. 状态管理: 每个子步骤调用 `comet-state checkpoint`

**产出**:
```
.claude/skills/devloop-build/SKILL.md
openspec/changes/<name>/design.md
openspec/changes/<name>/plan.md
openspec/changes/<name>/tasks.md
```

**验证**:
```bash
test -s .claude/skills/devloop-build/SKILL.md && \
grep -q 'comet-design\|writing-plans' .claude/skills/devloop-build/SKILL.md && \
echo "✅ T3.2 PASSED: devloop-build skill created (Design+Plan section)"
```

---

### T3.3 — 完善 devloop-build Skill（Build 部分）

**目标**: 编排 Comet Build 阶段，支持 executing-plans 和 subagent-driven-development 两种模式。

**输入**: T3.2

**实现步骤**:
1. 在 `.claude/skills/devloop-build/SKILL.md` 中添加 Build 部分
2. Build 模式选择:
   - 小型需求（< 5 tasks）→ `executing-plans`（主会话执行）
   - 中大型需求 → `subagent-driven-development`（后台子代理）
3. 每个 Task 执行循环:
   - 实现 → TDD 验证 → Spec Compliance Review → Code Quality Review
   - 通过 → Git commit（task 级别）
   - 失败 → 修复（最多 3 轮）
4. 全部 Task 完成后:
   - 门禁 G3.4: 所有 task checkbox 勾选、双审查通过、测试不降级、lint 零错误
   - `comet-guard --apply`

**产出**:
```
.claude/skills/devloop-build/SKILL.md    (Build 部分补全)
```

**验证**:
```bash
grep -q 'executing-plans\|subagent-driven-development' .claude/skills/devloop-build/SKILL.md && \
grep -q 'task.*checkbox\|双审查\|TDD' .claude/skills/devloop-build/SKILL.md && \
echo "✅ T3.3 PASSED: devloop-build skill Build section complete"
```

---

### T3.4 — 阶段三端到端测试

**目标**: 使用一个真实需求，走完完整的 Design→Plan→Build 流程。

**输入**: T3.3；Phase 2 产出的需求文档

**实现步骤**:
1. 选取 Phase 2 产出的一个需求（status: proposed）
2. 加载 `devloop-build` Skill
3. 走完 Design（确认设计方案）→ Plan（确认实现计划）→ Build（执行实现）
4. 验证代码变更通过所有测试
5. 验证门禁 G3.2-G3.4
6. Git commit

**验证**:
```bash
# 检查产物
echo "=== Design Doc ===" && test -s openspec/changes/*/design.md && echo "✓"
echo "=== Plan Doc ===" && test -s openspec/changes/*/plan.md && echo "✓"
echo "=== Tasks ===" && test -s openspec/changes/*/tasks.md && echo "✓"
echo "=== Git log ===" && git log --oneline --grep="\[devloop\] stage:3" -3

# 运行门禁
.comet/scripts/devloop-guard check stage-3
echo "✅ T3.4 PASSED: Phase 3 end-to-end test complete"
```

---

## Phase 4: 阶段四+五 — 验证交付与闭环

> **目标**: 实现验证交付和闭环回写，让系统能自动触发下一个需求。
> **完成标志**: 一个需求从进入阶段四到完成并触发下一个需求的全流程跑通。

---

### T4.1 — 创建 devloop-verify Skill（验证 + 修复循环）

**目标**: 编排阶段四的自动化验证、代码审查、问题修复循环。

**输入**: Phase 3 完成（有代码变更产物）

**实现步骤**:
1. 创建 `.claude/skills/devloop-verify/SKILL.md`
2. Skill 逻辑:
   - `comet-state scale <name>` 确定验证级别
   - 加载 `comet-verify`（已有 Skill）运行测试 + 构建 + lint
   - 门禁 G4.1: 全部通过？
   - 失败: 加载 `systematic-debugging`（已有 Skill）→ 修复 → 重试（最多 3 次）
   - 加载 `code-review`（已有 Skill）进行代码审查
   - 门禁 G4.2: 无 Critical 问题
   - 修复循环: 不超过 3 轮
   - 加载 `verification-before-completion`（已有 Skill）
   - 门禁 G4.4: 最终门禁
3. 验证策略: 聚焦输出正确性，不要求环境一致性（设计第 15 节）

**产出**:
```
.claude/skills/devloop-verify/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/devloop-verify/SKILL.md && \
grep -q 'comet-verify\|code-review\|systematic-debugging' .claude/skills/devloop-verify/SKILL.md && \
echo "✅ T4.1 PASSED: devloop-verify skill created"
```

---

### T4.2 — 完善 devloop-verify Skill（提交 + 需求状态更新）

**目标**: 验证通过后自动 Commit 代码、更新需求状态、移动需求文档。

**输入**: T4.1

**实现步骤**:
1. 在 `devloop-verify/SKILL.md` 中添加提交与状态更新部分
2. 提交流程:
   - Git commit 代码变更: `[devloop] stage:4 action:verify req:<id> ✓`
   - 更新需求文档 frontmatter: `status: completed`
   - Git mv: `requirements/in-progress/<id>.md` → `requirements/completed/<id>.md`
   - Git commit 需求状态: `[devloop] stage:5 action:complete req:<id> ✓`
3. 触发阶段五: 扫描下一个待处理需求

**产出**:
```
.claude/skills/devloop-verify/SKILL.md    (提交部分补全)
```

**验证**:
```bash
grep -q 'git commit.*stage:4\|git mv.*completed' .claude/skills/devloop-verify/SKILL.md && \
echo "✅ T4.2 PASSED: devloop-verify commit section complete"
```

---

### T4.3 — 创建 devloop-loop Skill（闭环调度）

**目标**: 实现阶段五的闭环逻辑：扫描队列→选取需求→触发下一阶段→监听新需求。

**输入**: T4.2

**实现步骤**:
1. 创建 `.claude/skills/devloop-loop/SKILL.md`
2. Skill 逻辑:
   - 扫描 `requirements/in-progress/` 中非完成/阻塞的需求
   - 门禁 G5.1: 状态一致性（frontmatter status 与文件位置一致）
   - 按优先级排序
   - 如果队列非空:
     - 取最高优先级需求
     - 根据状态路由: `proposed`→阶段二, `designed`→阶段三(Plan), `planned`→阶段三(Build)
   - 如果队列为空:
     - 输出状态摘要
     - 提示可以手动输入新需求或等待
3. 支持 `/devloop start` 命令触发

**产出**:
```
.claude/skills/devloop-loop/SKILL.md
```

**验证**:
```bash
test -s .claude/skills/devloop-loop/SKILL.md && \
grep -q '扫描\|优先级\|队列' .claude/skills/devloop-loop/SKILL.md && \
echo "✅ T4.3 PASSED: devloop-loop skill created"
```

---

### T4.4 — 阶段四端到端测试（单需求验证→完成）

**目标**: 使用 Phase 3 产出的代码变更，走完验证→提交→状态更新的完整流程。

**输入**: T4.2；Phase 3 产出的代码变更

**验证**:
```bash
# 1. 检查需求文档已移到 completed/
ls requirements/completed/REQ-*.md

# 2. 检查需求状态
grep 'status: completed' requirements/completed/REQ-*.md

# 3. 检查 Git commit
git log --oneline --grep="\[devloop\] stage:4" -1
git log --oneline --grep="\[devloop\] stage:5" -1

echo "✅ T4.4 PASSED: Single requirement verify→complete flow works"
```

---

### T4.5 — 阶段五端到端测试（闭环触发下一需求）

**目标**: 验证阶段四完成后自动检测并触发下一个需求。

**输入**: T4.3 + T4.4；至少有两个需求在队列中

**验证**:
```bash
# 1. 确认有多个待处理需求
PENDING=$(ls requirements/in-progress/REQ-*.md 2>/dev/null | wc -l)
echo "Pending requirements: $PENDING"

# 2. 运行 /devloop start 或加载 devloop-loop Skill
# 3. 验证自动选取了下一个最高优先级需求
# 4. 验证 loop-state.yaml 中 active_requirement 已更新

# 检查 active_requirement 更新
.comet/scripts/comet-state get active_requirement.id

echo "✅ T4.5 PASSED: Loop back to next requirement works"
```

---

## Phase 5: 防御与保鲜

> **目标**: 实现 KB 防御机制、漂移检测、定时保鲜和确认队列系统。
> **完成标志**: 漂移检测报告能正确识别过时 KB 条目，定时保鲜可触发。

---

### T5.1 — 实现 KB 漂移检测脚本

**目标**: 创建 `kb-drift-check` 脚本，对比 codegraph 当前符号信息与 KB 条目，生成漂移报告。

**输入**: Phase 1 完成（KB 可用）；Phase 0（comet-state 可用）

**实现步骤**:
1. 创建 `.comet/scripts/kb-drift-check` Bash 脚本
2. 脚本逻辑:
   - 读取 `knowledge-base/.manifest.yaml` 获取模块列表
   - 对每个模块: 用 codegraph 获取当前符号 → 对比 KB 条目中的 API/数据模型描述
   - 标记差异: api_signature_changed, new_entity, removed_function
   - 计算 drift_score（变更数 / 总条目数）
   - 生成 `knowledge-base/.drift-report.yaml`
3. 输出格式: 每个模块一行摘要 + 详细报告文件

**产出**:
```
.comet/scripts/kb-drift-check       (可执行脚本)
knowledge-base/.drift-report.yaml   (首次运行生成)
```

**验证**:
```bash
chmod +x .comet/scripts/kb-drift-check

# 运行漂移检测
.comet/scripts/kb-drift-check

# 验证报告存在且包含 drift_score
test -s knowledge-base/.drift-report.yaml && \
grep -q 'drift_score' knowledge-base/.drift-report.yaml && \
echo "✅ T5.1 PASSED: Drift detection works"
```

---

### T5.2 — 实现 KB 保鲜触发机制

**目标**: 实现阶段四后的自动保鲜 + 定时巡检 + 手动触发。

**输入**: T5.1

**实现步骤**:
1. 创建 `.comet/scripts/kb-freshness-trigger` Bash 脚本
2. 实现三种触发模式:
   - `--mode post-verify`: 阶段四后调用，只更新受影响模块
   - `--mode cron --action report`: 全量扫描，只生成报告
   - `--mode cron --action fix`: 全量扫描，自动修复 auto_fixable=true 的项
   - `--mode manual --scope <module|all> --action <report|fix>`: 手动触发
3. Git diff 检测: 找到上次 `[devloop] stage:1` commit，diff 到 HEAD 获取变更文件
4. 映射变更文件→受影响模块→增量更新
5. 用 `comet-state` 记录保鲜操作

**产出**:
```
.comet/scripts/kb-freshness-trigger    (可执行脚本)
```

**验证**:
```bash
chmod +x .comet/scripts/kb-freshness-trigger

# 测试手动触发（report 模式）
.comet/scripts/kb-freshness-trigger --mode manual --scope all --action report

# 验证输出
test -s knowledge-base/.drift-report.yaml && \
echo "✅ T5.2 PASSED: Freshness trigger works"
```

---

### T5.3 — 实现确认队列管理

**目标**: 创建确认队列系统，支持批量确认和超时处理。

**输入**: T0.4（comet-state 可用）

**实现步骤**:
1. 创建 `.comet/scripts/confirmation-queue` Bash 脚本
2. 实现子命令:
   - `confirmation-queue list` — 列出所有待确认项
   - `confirmation-queue show <id>` — 显示单个确认项详情
   - `confirmation-queue confirm <id>` — 确认
   - `confirmation-queue reject <id> "<reason>"` — 拒绝
   - `confirmation-queue skip <id> "<reason>"` — 跳过（需 Level 1 权限）
   - `confirmation-queue batch-confirm <id1,id2,...>` — 批量确认
   - `confirmation-queue check-timeout` — 检查超时项（>24h）
3. 确认项存储在 `loop-state.yaml` 的 `pending_confirmations` 中
4. 支持确认级别验证: Level 3 的项不能被 skip

**产出**:
```
.comet/scripts/confirmation-queue    (可执行脚本)
```

**验证**:
```bash
chmod +x .comet/scripts/confirmation-queue

# 测试生命周期
.comet/scripts/confirmation-queue list  # 应该列出任何待确认项或显示 "none"

echo "✅ T5.3 PASSED: Confirmation queue works"
```

---

### T5.4 — 更新 KB 条目信任级别

**目标**: 实现 KB 条目的信任级别自动评估和升级/降级逻辑。

**输入**: T5.1；KB 条目带有 frontmatter 防御标记

**实现步骤**:
1. 创建 `.comet/scripts/kb-trust-update` Bash 脚本
2. 脚本逻辑:
   - 扫描所有 KB 条目 frontmatter
   - 计算当前 trust level:
     - `confidence >= 0.9 AND drift_score < 0.05 AND source == 'human-reviewed'` → Trusted
     - `confidence >= 0.7 AND drift_score < 0.2` → Tentative
     - 其余 → Untrusted
   - 检测阻断条件:
     - 任意模块 drift_score > 0.3 → 输出阻断警告
     - 全局平均 drift_score > 0.2 → 输出阻断警告
     - 低 confidence (<0.7) 条目被 >3 需求引用 → 输出阻断警告
3. 更新 `.manifest.yaml` 中的 trust 摘要

**产出**:
```
.comet/scripts/kb-trust-update    (可执行脚本)
```

**验证**:
```bash
chmod +x .comet/scripts/kb-trust-update

# 运行信任评估
.comet/scripts/kb-trust-update

# 验证输出包含信任级别统计
echo "✅ T5.4 PASSED: Trust level update works"
```

---

### T5.5 — Phase 5 端到端测试（保鲜 + 防御联动）

**目标**: 验证漂移检测→信任评估→保鲜修复的全链路。

**验证**:
```bash
# 1. 运行漂移检测
.comet/scripts/kb-drift-check

# 2. 运行信任更新
.comet/scripts/kb-trust-update

# 3. 模拟代码变更后运行保鲜
# (在 Claude Code 中修改一个模块的代码，然后:)
.comet/scripts/kb-freshness-trigger --mode manual --scope all --action report

# 4. 检查漂移报告是否正确反映了变更
echo "=== Drift Report ===" && head -30 knowledge-base/.drift-report.yaml

# 5. 验证阻断逻辑
# 如果漂移超过阈值，检查是否输出阻断警告
.comet/scripts/kb-trust-update 2>&1 | grep -q "阻断\|BLOCK" && echo "阻断逻辑正常" || echo "(漂移未超阈值，阻断未触发)"

echo "✅ T5.5 PASSED: Defense & freshness integration works"
```

---

## Phase 6: 端到端集成测试

> **目标**: 使用 3 个不同类型/优先级的需求，验证完整的 5 阶段闭环能正确运行。
> **完成标志**: 3 个需求全部完成，每个都经历了完整的 5 阶段流程。

---

### T6.1 — 集成测试：Bugfix 需求（Level 1 快速通道）

**目标**: 用一个简单 bugfix 验证 Level 1 快速通道（零人工确认直到阶段四）。

**测试用例**: 从代码中选择一个简单 bug（如 typo fix 或逻辑小错误），作为需求输入。

**验证**:
```bash
# 验证结果
echo "=== 需求状态 ===" && grep 'status: completed' requirements/completed/REQ-*.md
echo "=== 类型 ===" && grep 'type: bugfix' requirements/completed/REQ-*.md
echo "=== 确认级别 ===" && grep 'confirmation_level' requirements/completed/REQ-*.md
echo "=== Git 链 ===" && git log --oneline --grep="req:" | head -10
echo "✅ T6.1 PASSED: Bugfix fast-track works"
```

---

### T6.2 — 集成测试：Feature 需求（Level 3 高保障）

**目标**: 用一个中型 feature 需求验证 Level 3 高保障（包含所有确认点）。

**测试用例**: 一个涉及 2-3 个模块的新功能需求。

**验证**:
```bash
# 验证所有确认点都有记录
.comet/scripts/comet-state get pending_confirmations
# 验证 design.md, plan.md, tasks.md 都存在
# 验证最终代码变更通过所有测试
echo "✅ T6.2 PASSED: Feature high-assurance works"
```

---

### T6.3 — 集成测试：闭环连续运行（2 个需求自动流转）

**目标**: 验证第一个需求完成后自动触发第二个需求（闭环）。

**测试用例**: 准备两个需求，第一个完成后验证第二个自动启动。

**验证**:
```bash
# 1. 确认第一个需求已完成
grep 'status: completed' requirements/completed/REQ-*.md

# 2. 确认第二个需求已被选取
.comet/scripts/comet-state get active_requirement.id

# 3. 确认 loop-state 记录了两个需求的流转
.comet/scripts/comet-state get completed_steps

echo "✅ T6.3 PASSED: Closed-loop auto-transition works"
```

---

### T6.4 — 中断恢复测试

**目标**: 模拟会话中断，验证恢复流程能正确从 checkpoint 继续。

**测试场景**:
1. 启动一个需求的处理流程
2. 在阶段三 Design 完成后（确认点前）模拟中断（修改 loop-state.yaml 模拟未完成状态）
3. 运行恢复: `comet-state check <name> build --recover`
4. 加载 `devloop-resume` Skill
5. 验证从 Design 确认点恢复继续

**验证**:
```bash
# 1. 模拟中断状态
.comet/scripts/comet-state init recovery-test
.comet/scripts/comet-state set current_stage 3
.comet/scripts/comet-state checkpoint 3 3.3 "design.md created, awaiting confirmation"
.comet/scripts/comet-state add-pending-confirmation plan_ready REQ-TEST "等待确认设计方案"

# 2. 运行恢复
.comet/scripts/comet-state check recovery-test build --recover

# 3. 验证恢复动作正确（应提示处理待确认项）
echo "✅ T6.4 PASSED: Interrupt recovery works"
```

---

## 附录 A: 任务依赖关系

```
T0.1 ──→ T0.2 ──→ T0.3
  │        │
  └──→ T0.4 ──→ T0.6
  │
  └──→ T0.5 ──→ T0.7

Phase 0 ──→ T1.1 ──→ T1.2 ──→ T1.3 ──→ T1.4 ──→ T1.5 ──→ T1.6
                                                                    │
Phase 1 ──→ T2.1 ──→ T2.2 ──→ T2.3 ──→ T2.4 ──→ T2.5              │
                                                                    │
Phase 2 ──→ T3.1 ──→ T3.2 ──→ T3.3 ──→ T3.4                       │
                                                                    │
Phase 3 ──→ T4.1 ──→ T4.2 ──→ T4.3 ──→ T4.4 ──→ T4.5              │
                                                                    │
Phase 4+ ──→ T5.1 ──→ T5.2 ──→ T5.3 ──→ T5.4 ──→ T5.5             │
                                                                    │
Phase 5 ──→ T6.1 ──→ T6.2 ──→ T6.3 ──→ T6.4 ──────────────────────┘
```

## 附录 B: Skill 文件清单

### 新创建的 Skill

| Skill 名称 | 任务 | 功能 |
|-----------|------|------|
| `devloop-resume` | T0.6 | 中断恢复 |
| `kb-module-extract` | T1.1 | 单模块 KB 条目生成 |
| `kb-code-scan` | T1.2 | 代码扫描 + manifest 生成 |
| `kb-aggregate` | T1.4 | KB 汇总整合 |
| `req-reverse-infer` | T1.5 | 需求反向推断 |
| `devloop-reverse` | T1.6 | 阶段一总编排 |
| `req-parse-input` | T2.1 | 需求输入解析 |
| `kb-indexed-load` | T2.2 | 索引式 KB 加载 |
| `req-impact-analysis` | T2.3 | 影响分析 |
| `req-generate-doc` | T2.4 | 需求文档生成 |
| `devloop-intake` | T2.5 | 阶段二总编排 |
| `req-select-next` | T3.1 | 需求拉取与选取 |
| `devloop-build` | T3.2/3.3 | 阶段三总编排 |
| `devloop-verify` | T4.1/4.2 | 阶段四验证交付 |
| `devloop-loop` | T4.3 | 阶段五闭环调度 |

### 新创建的脚本

| 脚本路径 | 任务 | 功能 |
|---------|------|------|
| `.comet/scripts/comet-state` | T0.4 | 状态管理 |
| `.comet/scripts/devloop-guard` | T0.5 | 门禁执行引擎 |
| `.comet/scripts/kb-drift-check` | T5.1 | KB 漂移检测 |
| `.comet/scripts/kb-freshness-trigger` | T5.2 | KB 保鲜触发 |
| `.comet/scripts/confirmation-queue` | T5.3 | 确认队列管理 |
| `.comet/scripts/kb-trust-update` | T5.4 | KB 信任级别评估 |

## 附录 C: 每个任务对 Claude Code 的使用方式

所有任务设计为可以通过以下方式交由 Claude Code 执行：

```
# 方式 1: 直接描述任务
"请执行 T0.1: 创建 DevLoop 目录结构"

# 方式 2: 引用任务文件
"请按照 design/05-implementation-tasks.md 中的 T1.3 执行"

# 方式 3: 加载 Skill 后执行
"加载 kb-module-extract Skill，对 src/auth 模块生成 KB 条目"
```

每个任务完成后，Claude Code 会:
1. 生成产物文件
2. 运行验证命令
3. 报告通过/失败
4. 如果失败，修复后重新验证
