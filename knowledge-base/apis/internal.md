---
source: auto-generated
confidence: 0.90
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - knowledge-base/modules/comet-scripts/api.md
  - knowledge-base/modules/devloop-skills/api.md
  - knowledge-base/modules/kb-templates/api.md
  - knowledge-base/modules/design-docs/api.md
---

# DevLoop 内部 API 总览

## comet-scripts → devloop-skills 接口

### comet-state 子命令（被所有编排 Skill 调用）

| 命令 | 用途 | 调用阶段 |
|------|------|---------|
| `init <change-name>` | 初始化状态文件 | 各阶段开始 |
| `get <key>` | 读取字段值 | 各阶段 |
| `set <key> <value>` | 设置字段值 | 各阶段 |
| `checkpoint <stage> <step> <desc>` | 保存断点 | 每 Step 结束 |
| `complete-step <stage> <step>` | 标记步骤完成 | 阶段完成 |
| `gate-result <id> <pass\|fail>` | 记录门禁结果 | 门禁校验后 |
| `add-pending <type> <req> <msg>` | 添加待确认项 | 确认点 |
| `list-pending` | 列出待确认项 | 中断恢复 |
| `check <name> <phase> --recover` | 恢复检查 | 中断恢复 |
| `scale [name]` | 显示状态摘要 | 阶段四 |

### devloop-guard 子命令（被所有编排 Skill 调用）

| 命令 | 用途 | 调用阶段 |
|------|------|---------|
| `check <stage>` | 运行硬门禁检查 | 每阶段结束 |
| `list [--type <t>]` | 列出门禁清单 | 调试 |
| `validate-config` | 验证配置文件 | Phase 0 |

## devloop-skills 间调用接口

### 编排 Skill → 通用 Skill

| 调用方 | 被调用方 | 输入 | 产出 |
|--------|---------|------|------|
| devloop-reverse | dispatching-parallel-agents | 模块清单 + 分析提示词 | 每个模块 5 KB 文件 |
| devloop-reverse | brainstorming | 汇总 KB | 基线需求 |
| devloop-intake | brainstorming | 需求 + KB 上下文 | 影响分析 |
| devloop-intake | openspec-propose | 需求文档 | OpenSpec Change |
| devloop-build | brainstorming | 需求 + KB 上下文 | design.md |
| devloop-build | writing-plans | design.md | plan.md + tasks.md |
| devloop-build | subagent-driven-development | tasks.md + 上下文 | 代码变更 |
| devloop-build | executing-plans | tasks.md + 上下文 | 代码变更 |
| devloop-verify | verification-before-completion | 代码变更 | 验证结果 |
| devloop-verify | code-review | 代码变更 | 审查报告 |
| devloop-verify | systematic-debugging | 失败信息 | 修复方案 |
| devloop-verify | finishing-a-development-branch | 代码变更 | 合并结果 |

### sdd 内部调用链

```
subagent-driven-development
  ├── test-driven-development (每个 task)
  └── requesting-code-review (每个 task)
       └── receiving-code-review (审查者返回意见)
```

## 模板接口（kb-templates → devloop-skills）

各 Skill 读取模板时通过以下"契约"：

```
模板文件 = frontmatter YAML + Markdown 正文 + <!-- Variables: --> 变量清单
```

消费方 LLM 读取模板后：
1. 解析 frontmatter 获取元数据
2. 理解 Markdown 结构和变量占位符 `{{ }}`
3. 从分析上下文中提取对应值填充变量
4. 用填充后的内容写入目标文件

### 模板消费关系

| 模板 | 消费方 | 消费阶段 |
|------|--------|---------|
| kb-module-*.md (5个) | devloop-reverse Step 2 | 阶段一 |
| requirement-input.md | devloop-intake Step 1 | 阶段二 |
| requirement-doc.md | devloop-intake Step 5 | 阶段二 |
| impact-analysis.md | devloop-intake Step 3 | 阶段二 |
| nfr-checklist.md | devloop-intake Step 4 | 阶段二 |
| verification-checklist.md | devloop-verify Step 1 | 阶段四 |
| kb-manifest.yaml | devloop-reverse Step 1 | 阶段一 |

## MCP 工具接口

| MCP 工具 | 调用方 | 用途 |
|----------|--------|------|
| `codegraph_explore` | devloop-reverse, devloop-intake | 项目结构扫描、KB 索引查询 |
| `codegraph_node` | devloop-reverse (agent 内部) | 单模块符号/API/依赖读取 |
