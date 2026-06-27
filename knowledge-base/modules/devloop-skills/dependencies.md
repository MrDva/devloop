---
module: devloop-skills
source: auto-generated
confidence: 0.80
drift_score: 0.0
last_updated: "2026-06-27T14:00:00Z"
generated_from:
  - .claude/skills/
---

# devloop-skills -- 依赖关系

## 上游依赖（本模块依赖的）

### comet-scripts
- **依赖类型**: 脚本运行时
- **依赖内容**: `comet-state` 状态管理脚本、`devloop-guard` 门禁校验脚本
- **耦合度**: 高（devloop-reverse 和 devloop-resume 直接调用）
- **接口**: `.comet/scripts/comet-state checkpoint|check|set|list-pending`; `.comet/scripts/devloop-guard check stage-1`

### kb-templates
- **依赖类型**: 模板引用
- **依赖内容**: `templates/kb-module-*.md` 系列模板
- **耦合度**: 中（devloop-reverse Step2 使用模板生成 KB 文件）
- **接口**: `templates/kb-module-overview.md`, `kb-module-api.md`, `kb-module-data-model.md`, `kb-module-business-logic.md`, `kb-module-dependencies.md`

### codegraph（MCP）
- **依赖类型**: MCP 工具
- **依赖内容**: `codegraph_explore`（全项目扫描）、`codegraph_node`（单文件/符号读取）
- **耦合度**: 高（devloop-reverse Step1 和 Step2 的核心扫描工具）
- **接口**: `codegraph_explore`（自然语言查询 + 返回源代码）; `codegraph_node file=<path>`（返回文件内容 + 行号）

### OpenSpec CLI
- **依赖类型**: CLI 工具
- **依赖内容**: `openspec` 命令（new, list, status, instructions, archive 等）
- **耦合度**: 高（所有 11 个 openspec-* Skill 依赖此 CLI）
- **接口**: `openspec new change <name>`, `openspec list --json`, `openspec status --change <name> --json`, `openspec instructions apply --change <name> --json`, `openspec archive change <name>`

## 下游依赖（依赖本模块的）

### 无
- **依赖类型**: 无
- **依赖内容**: Skill 是顶层编排方，不被其他模块调用
- **耦合度**: 无

## 依赖图

```
                          +------------------+
                          |  devloop-skills  |
                          +--------+---------+
                                   |
            +----------------------+-----------------------+
            |                      |                       |
            v                      v                       v
   +---------------+      +------------------+     +------------------+
   | comet-scripts |      |  kb-templates    |     |    codegraph     |
   | (comet-state, |      | (module templates)|    | (MCP tools:      |
   |  devloop-guard)|     +------------------+     |  explore, node)  |
   +---------------+                                +------------------+
                                                            |
                                                            v
                                                  +------------------+
                                                  |  OpenSpec CLI    |
                                                  | (openspec cmd)   |
                                                  +------------------+


                         Skill 间横向调用关系图:

+--------------------+       +-------------------------+
| devloop-reverse    | ----> | dispatching-parallel-   |
| (阶段一编排)        |       | agents                  |
+--------------------+       +-------------------------+
         |                              |
         v                              v
+--------------------+       +-------------------------+
| brainstorming      |       | codegraph_node (MCP)   |
| (反推基线需求)      |       | (每个子 agent 读取代码)  |
+--------------------+       +-------------------------+

+--------------------+       +-------------------------+       +-------------------------+
| devloop-build      | ----> | subagent-driven-        | ----> | requesting-code-review  |
| (阶段三编排)        |       | development (SDD模式)    |       | (spec + quality 审查)   |
+--------------------+       +-------------------------+       +-------------------------+
         |                              |
         |       +----------------------+
         |       v
         |  +-------------------------+       +-------------------------+
         |  | systematic-debugging    |       | executing-plans         |
         |  | (修复循环时加载)         |       | (EP降级模式)              |
         |  +-------------------------+       +-------------------------+
         |                                              |
         |                                              v
         |                                     +-------------------------+
         |                                     | finishing-a-development-|
         |                                     | branch                  |
         |                                     +-------------------------+
         |                                              |
         |                                              v
         |                                     +-------------------------+
         |                                     | verification-before-    |
         |                                     | completion              |
         |                                     +-------------------------+

+--------------------+       +-------------------------+
| devloop-verify     | ----> | verification-before-    |
| (阶段四编排)        |       | completion              |
+--------------------+       +-------------------------+

+--------------------+       +-------------------------+
| devloop-resume     | ----> | 根据 stage 调用         |
| (中断恢复)          |       | reverse/intake/build/  |
+--------------------+       | verify/loop             |
                             +-------------------------+

+-------------------------+       +-------------------------+
| writing-skills          | ----> | test-driven-development |
| (元技能: 创建Skill)     |       | (必须理解 TDD 循环)     |
+-------------------------+       +-------------------------+

OpenSpec 系列 Skill (11 个) 相互独立，均依赖 openspec CLI:
    openspec-new-change
    openspec-propose     ---> openspec-new-change (间接)
    openspec-explore     (独立)
    openspec-ff-change   ---> openspec-new-change (快速路径)
    openspec-continue-change (独立)
    openspec-apply-change (独立)
    openspec-verify-change (独立)
    openspec-sync-specs  (独立)
    openspec-archive-change (独立)
    openspec-bulk-archive-change (独立)
    openspec-onboard     (独立)
```

## 循环依赖

- **DevLoop 阶段循环**: 阶段五（Loop）决策后可能回到阶段一（Reverse）或阶段二（Intake），这是设计预期的流程循环，不是代码层面的循环依赖
- **无代码级别循环依赖**: Skill 间调用关系为 DAG（有向无环图）

---
