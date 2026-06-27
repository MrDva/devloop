---
req_id: {{ req_id }}
req_title: {{ title }}
verification_date: {{ verification_date }}
verified_by: {{ verified_by }}
---

# 验证检查清单: {{ title }}

## 自动化验证

### 单元测试
- [ ] 所有单元测试通过（退出码=0）
- [ ] 新增代码有对应的单元测试
- [ ] 关键路径测试覆盖率满足要求

### 集成测试
- [ ] 所有集成测试通过（退出码=0）
- [ ] 模块间接口契约验证通过

### 构建
- [ ] 构建成功（退出码=0）
- [ ] Lint 零错误
- [ ] Type Check 零错误

## 验收标准验证

{{#each acceptance_criteria}}
- [ ] {{ this }}
{{/each}}

## 非功能需求验证

{{#each nfr_checks}}
### {{ this.dimension }}
- [ ] {{ this.check_item }}
{{/each}}

## 代码审查

- [ ] 审查报告存在
- [ ] 无 Critical 级别问题
- [ ] High 级别问题已处理
- [ ] 与 KB 记录的风格一致

## KB 一致性

- [ ] 代码变更未破坏 KB 中记录的 API 契约
- [ ] 新增 API 已在 KB 中有对应条目（或已标记待更新）
- [ ] 移除的 API 已在 KB 中标注

## 修复循环

- 修复次数: {{ fix_attempts }}
- 修复循环限制: ≤ 3 次
- 每次修复后未引入新 Critical 问题

## 门禁结果

| 门禁 | 结果 | 备注 |
|------|------|------|
| G4.1 自动化验证 | {{ g4_1_result }} | {{ g4_1_note }} |
| G4.2 代码审查 | {{ g4_2_result }} | {{ g4_2_note }} |
| G4.3 修复循环 | {{ g4_3_result }} | {{ g4_3_note }} |
| G4.4 阶段四总门禁 | {{ g4_4_result }} | {{ g4_4_note }} |

## 最终判定

- [ ] 所有验收标准已满足
- [ ] 所有门禁通过
- [ ] 可以完成交付

---
<!-- Variables:
  req_id / title — the requirement being verified
  verification_date — ISO 8601 timestamp
  verified_by — "AI" | "human"
  acceptance_criteria — list of criteria from requirement doc
  nfr_checks — list of {dimension, check_item}
  fix_attempts — number of fix cycles taken
  g4_1_result through g4_4_result — PASS | WARN | FAIL
  g4_1_note through g4_4_note — notes
-->
