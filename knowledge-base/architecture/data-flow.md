---
source: auto-generated
confidence: 0.90
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - knowledge-base/modules/comet-scripts/
  - knowledge-base/modules/devloop-skills/
  - knowledge-base/modules/kb-templates/
  - knowledge-base/modules/design-docs/
---

# DevLoop 数据流

## 五阶段主数据流

```
[源代码] ──→ 阶段一 (reverse) ──→ knowledge-base/ (KB 条目)
                                      │
                        ┌─────────────┘
                        ▼
                  requirements/baseline/ (基线需求)
                        
[用户需求输入] ──→ 阶段二 (intake) ──→ requirements/in-progress/REQ-*.md
                                           │
                                    ┌──────┘
                                    ▼
                              openspec/changes/<name>/
                              (proposal.md + .comet.yaml)
                                           │
[需求文档 + KB] ──→ 阶段三 (build) ──→ openspec/changes/<name>/
                                           │  design.md + plan.md + tasks.md
                                    ┌──────┘
                                    ▼
                              src/ (代码变更)
                                           │
[代码变更] ──→ 阶段四 (verify) ──→ 验证结果 + 审查结果
                                           │
                                    ┌──────┘
                                    ▼
                              requirements/completed/REQ-*.md
                              (归档 + git merge)
                                           │
[已完成需求] ──→ 阶段五 (loop) ──→ 下一需求 (回到阶段二或三)
```

## 关键数据实体变更

### loop-state.yaml（运行时状态）

由 `comet-state` 管理，贯穿所有阶段：

```
init → set current_stage → checkpoint (每步) → gate-result (门禁后) → complete-step (每步结束)
```

### guard-config.yaml（门禁配置）

静态配置，由 `devloop-guard` 在每阶段结束前读取并执行检查。

### KB 条目（knowledge-base/modules/）

阶段一创建，阶段四触发增量更新，阶段五的保鲜定时任务全量扫描：
- `drift_score` 随代码变更递增
- `confidence` 随人工审核而调整
- `last_updated` 记录最后更新时间

### 需求文档状态机

```
proposed (阶段二产出)
  → designed (阶段三 brainstorm 后)
  → planned (阶段三 writing-plans 后)
  → built (阶段三 sdd 完成后)
  → verified (阶段四验证后)
  → completed (阶段四归档后)
  
  异常状态:
  → blocked (依赖不满足)
  → rejected (被拒绝)
  → deferred (延期)
```

## 上下文窗口预算流（阶段二/三）

```
总预算: ~200K tokens
├── 系统指令 + Skill 定义: ~20K
├── 需求文档: ~5K
├── KB 相关条目 (索引按需加载): ~80K  ← 摘要层降级策略
├── 相关源代码 (codegraph 实时): ~50K
├── 生成产物: ~30K
└── 缓冲: ~15K
```

超出预算时触发摘要层降级：优先加载模块 `overview.md`，token 不足时使用 `.manifest.yaml` 中的 `summary_tokens_estimate` 字段预判并跳过完整 API 文档。
