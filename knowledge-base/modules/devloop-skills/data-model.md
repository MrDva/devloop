---
module: devloop-skills
source: auto-generated
confidence: 0.80
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - .claude/skills/
  - .comet/loop-state.yaml  (referenced)
---

# devloop-skills -- 数据模型

## 实体定义

### SKILL.md（Skill 定义文件）
- **类型**: Markdown 文件（含 YAML frontmatter）
- **描述**: 每个 Skill 的定义、描述、前置条件、流程步骤和门禁规则
- **字段**:
  - `name` (string) **必需**: Skill 名称，如 devloop-reverse
  - `description` (string) **必需**: Skill 功能摘要
  - `license` (string): 许可证信息（仅 OpenSpec Skill）
  - `compatibility` (string): 兼容性要求（仅 OpenSpec Skill）
  - `metadata.author` (string): 作者（仅 OpenSpec Skill）
  - `metadata.version` (string): 版本号（仅 OpenSpec Skill）
  - `metadata.generatedBy` (string): 生成工具版本（仅 OpenSpec Skill）
- **约束**:
  - 文件名必须为 `SKILL.md`
  - 必须位于 `.claude/skills/<name>/` 目录下
  - frontmatter 必须包含 `name` 和 `description`
- **关联**:
  - -> other SKILL.md: 通过内部调用关系引用（如 devloop-reverse 调用 brainstorming）

### loop-state.yaml（运行时状态文件）
- **类型**: YAML 文件
- **描述**: DevLoop 运行时状态，记录当前阶段、步骤、checkpoint 和待确认项。用于中断恢复和流程追踪
- **字段**:
  - `devloop.version` (string) **必需**: 状态文件版本号（当前 "1.0"）
  - `devloop.change_name` (string): 当前 change 名称
  - `devloop.current_stage` (integer) **必需**: 当前阶段编号（1-5）
  - `devloop.current_step` (string) **必需**: 当前步骤标识（如 "3.4"）
  - `devloop.active_requirement.id` (string): 当前活跃需求 ID
  - `devloop.active_requirement.title` (string): 当前活跃需求标题
  - `devloop.checkpoint.stage` (integer): checkpoint 所在阶段
  - `devloop.checkpoint.step` (string): checkpoint 所在步骤
  - `devloop.checkpoint.last_action` (string): 最后成功动作描述
  - `devloop.checkpoint.timestamp` (datetime): checkpoint 时间戳
  - `devloop.checkpoint.git_commit` (string): 关联 git commit SHA
  - `devloop.pending_confirmations[]` (array): 待确认项列表
  - `devloop.pending_confirmations[].type` (string): 确认类型（如 plan_ready）
  - `devloop.pending_confirmations[].req` (string): 关联需求 ID
  - `devloop.pending_confirmations[].message` (string): 确认提示信息
  - `devloop.errors[]` (array): 错误记录列表
- **约束**:
  - 位置固定为 `.comet/loop-state.yaml`
  - `current_stage` 范围为 1-5
- **关联**:
  - -> OpenSpec Change 目录: `change_name` 对应 `openspec/changes/<name>/`
  - -> state checkpoint: `checkpoint` 段用于中断恢复

### .manifest.yaml（模块清单）
- **类型**: YAML 文件
- **描述**: devloop-reverse Step1 生成的模块清单，列出项目所有模块
- **字段**:
  - `modules[].name` (string) **必需**: 模块名
  - `modules[].path` (string) **必需**: 源代码目录路径
  - `modules[].type` (string) **必需**: 模块类型（service/gateway/library/utility/ui/data/config/other）
  - `modules[].files` (integer): 文件数
  - `modules[].dependencies[]` (array): 依赖的其他模块名
  - `modules[].keywords[]` (array): 提取的关键词
  - `modules[].entities[]` (array): 核心实体名
  - `modules[].apis[]` (array): 公共 API 摘要
  - `modules[].total_tokens_estimate` (integer): 估计总 token 数
  - `modules[].summary_tokens_estimate` (integer): 仅 overview 的 token 数
- **约束**:
  - 位置固定为 `knowledge-base/.manifest.yaml`
  - `modules_analyzed / total_modules >= 0.8`（门禁 G1.1）
- **关联**:
  - -> KB 模块条目: `modules[].name` 对应 `knowledge-base/modules/<name>/`

### OpenSpec Change 目录（openspec/changes/<name>/）
- **类型**: 目录结构
- **描述**: 由 openspec CLI 创建，存储单个变更从创建到归档的全生命周期 artifact
- **字段**（子文件）:
  - `.openspec.yaml`: OpenSpec 元数据配置
  - `proposal.md`: 提案（what & why）
  - `design.md`: 设计文档（how）
  - `tasks.md`: 实现任务列表
  - `specs/`: delta spec 目录
- **约束**:
  - 路径模式 `openspec/changes/<kebab-case-name>/`
  - 归档后标记为 archive 状态
- **关联**:
  - -> loop-state.yaml: `change_name` 引用此目录

### 基线需求文档（requirements/baseline/REQ-<NNN>.md）
- **类型**: Markdown 文件（含 YAML frontmatter）
- **描述**: devloop-reverse Step4 通过 brainstorming 从 KB 反推的基线需求
- **字段**:
  - `id` (string) **必需**: 需求 ID（REQ-<NNN>）
  - `title` (string) **必需**: 需求标题
  - `module` (string) **必需**: 关联模块
  - `status` (string) **必需**: 固定为 `baseline`
  - `inferred_from[]` (array) **必需**: 来源文件路径
  - `confidence` (string) **必需**: high/medium/low
  - `source` (string) **必需**: 固定为 `auto-inferred`
  - `usage_restriction` (string) **必需**: 固定为 `reference_only`
  - `created` (datetime) **必需**: 创建时间
- **约束**:
  - `usage_restriction: reference_only` 必须保留（防止被用作阶段三强制约束）
  - `covered_modules / total_modules >= 0.9`（门禁 G1.4）
  - `low confidence` 需求占比 <= 20%（门禁 G1.4）
- **关联**:
  - -> KB 模块条目: `module` 关联到 `knowledge-base/modules/<module>`

### KB 模块条目（knowledge-base/modules/<name>/）
- **类型**: 目录结构，含 5 个 Markdown 文件
- **描述**: 每个模块的深度分析产物
- **文件**:
  - `overview.md`: 职责、核心功能、对外接口摘要、依赖拓扑
  - `api.md`: 导出符号完整签名、参数、返回值、异常
  - `data-model.md`: 数据结构、类型定义、实体关系
  - `business-logic.md`: 核心流程、业务规则、副作用
  - `dependencies.md`: 上下游依赖、耦合度、循环依赖检测
- **字段**（每个文件 frontmatter）:
  - `module` (string): 模块名
  - `source` (string): 固定为 `auto-generated`
  - `confidence` (float): 0.0-1.0
  - `drift_score` (float): 初始 0.0
  - `last_updated` (datetime): 更新时间
  - `generated_from[]` (array): 源代码文件路径追溯
- **约束**:
  - 每个模块必须有至少 `overview.md` + `api.md`（门禁 G1.2）
  - 单模块上下文 < 50K tokens（超大模块拆分子模块）
  - `module-name` wiki link 引用目标必须存在（门禁 G1.3）

### 门禁结果文件（.comet/gate-results/soft-<gate-id>.json）
- **类型**: JSON 文件
- **描述**: 软门禁评估结果，由 LLM agent 逐项评估后写入
- **字段**: 动态，取决于门禁类型

## 状态转换

### DevLoop 阶段转换
```
[Reverse(1)] -> [Intake(2)] -> [Build(3)] -> [Verify(4)] -> [Loop(5)]
                                                                |
                                                                v
                                                   [Reverse(1)] 或 [结束]
```

### DevLoop Build 阶段修复循环（最大 3 轮）
```
[Task Failed] -> [systematic-debugging] -> [Fix Attempt] -> [Verify]
                                     ^                        |
                                     |                   [Failed x3?]
                                     |                        |
                                     +---- 重试 ( <3 ) ------+
                                                       |
                                                  [ >=3 次 -> PAUSE ]
```

---
