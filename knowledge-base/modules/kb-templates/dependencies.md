---
module: kb-templates
source: auto-generated
confidence: 1.0
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - templates/
---

# kb-templates — 依赖关系

## 上游依赖（本模块依赖的）

**无上游依赖。**

本模块是纯模板定义，所有文件是静态的 Markdown/YAML 格式定义，不包含可执行代码，不引用其他模块的任何 API 或数据结构。模板是自包含的——它们只定义占位符和格式，不依赖其他模块来自身生效。

---

## 下游依赖（依赖本模块的）

### 1. devloop-skills（编排 Skill）

- **依赖类型**: 引用（引用模板文件）
- **依赖内容**: 读取 templates/ 下的模板文件，填充变量后生成各阶段产出物
- **耦合度**: 松散耦合——devloop-skills 通过文件系统路径读取模板，模板路径和变量名称构成隐式契约
- **具体用途**:
  - `requirement-input.md` → 接收初始需求输入
  - `requirement-doc.md` → 生成完整需求文档
  - `impact-analysis.md` → 生成影响分析报告
  - `nfr-checklist.md` → 执行 NFR 强制评估
  - `verification-checklist.md` → 生成验证检查清单

### 2. devloop-reverse（Step 2）

- **依赖类型**: 引用（引用模板文件）
- **依赖内容**: Step 2 使用 `kb-module-*.md` 5 个模板生成 KB 条目
- **耦合度**: 松散耦合——按照模板变量清单填充，模板结构变更需要同步更新生成逻辑
- **具体用途**:
  - `kb-module-overview.md` → 生成模块概述 KB 条目
  - `kb-module-api.md` → 生成 API 文档 KB 条目
  - `kb-module-data-model.md` → 生成数据模型 KB 条目
  - `kb-module-business-logic.md` → 生成业务逻辑 KB 条目
  - `kb-module-dependencies.md` → 生成依赖关系 KB 条目
  - `kb-manifest.yaml` → 生成/更新模块清单索引

### 3. devloop-resume（恢复流程）

- **依赖类型**: 引用（引用模板文件）
- **依赖内容**: 恢复时读取 KB 条目模板确定状态
- **耦合度**: 松散耦合

---

## 依赖图

```
┌─────────────────────────────────────────────────┐
│                  kb-templates                    │
│              (11 templates/* 文件)               │
│                                                 │
│  ┌─ KB 条目模板 ──┐  ┌─ 需求模板 ───────────┐  │
│  │ kb-module-*  ×5 │  │ requirement-input.md │  │
│  │                 │  │ requirement-doc.md   │  │
│  └───────┬─────────┘  └──────┬───────────────┘  │
│          │                   │                    │
│  ┌───────┴───────────────────┴────────────┐      │
│  │  analysis/verification 模板 ×3         │      │
│  │  manifest 模板 ×1                      │      │
│  └────────────────────────────────────────┘      │
└────────────────┬────────────────────────────────┘
                 │ (文件系统读取)
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌────────┐ ┌──────────┐ ┌──────────┐
│reverse │ │  skills  │ │  resume  │
│ Step 2 │ │ 各阶段   │ │ 恢复流程 │
└────────┘ └──────────┘ └──────────┘
```

---

## 循环依赖

✅ 未检测到循环依赖

- kb-templates 不依赖任何其他模块
- 下游模块（devloop-skills、devloop-reverse）单向引用 kb-templates
- 不存在模板引用下游模块的场景

---

## 隐式契约说明

虽然本模块无上游依赖，但模板与下游消费者之间存在隐式契约：

### 变量名称契约
- 模板中的 `{{ variable_name }}` 变量名是下游填充代码必须理解和提供的字段
- 变量名变更需要同步更新所有下游消费者的填充逻辑
- 新变量添加不会破坏向后兼容（可选变量可省略）

### 文件路径契约
- 模板通过固定路径 `templates/<name>` 被引用
- 路径变更需要更新所有下游消费者的引用
- `knowledge-base/modules/<module_name>/` 是 KB 条目输出路径

### 格式结构契约
- frontmatter 格式（`---` 分隔的 YAML + Markdown 正文）
- Handlebars 风格的 `{{#each}}`、`{{#if}}` 控制结构
- 底部的 `<!-- Variables: -->` 注释作为变量清单声明
