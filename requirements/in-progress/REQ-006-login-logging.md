---
id: REQ-006
title: 添加用户登录日志功能
type: feature
priority: medium
status: proposed
module: auth-service
source: user-submitted
usage_restriction: none
kb_context:
  - knowledge-base/architecture/overview.md
confidence: low
created: 2026-06-27T07:12:00Z
updated: 2026-06-27T07:14:00Z
---

# REQ-006: 添加用户登录日志功能

> ⚠️ **置信度: low** — KB 无匹配认证/日志模块。影响分析基于通用推理。随 KB 扩展应重新评估。

## 背景

当前系统缺少登录行为的审计追踪能力。安全团队无法获知谁在何时从何处登录。添加登录日志功能可以记录每次登录的时间、来源 IP 和 User-Agent 信息，为安全审计和异常登录检测提供数据基础。

## 功能描述

系统在每次用户登录成功后自动记录登录日志，包含：
- 登录时间戳（精确到秒）
- 来源 IP 地址（IPv4/IPv6）
- User-Agent 字符串
- 登录是否成功

提供管理端 API 按时间范围、用户等条件查询登录日志。日志保留 90 天自动清理。

## 涉及模块

- **auth-service**: 在登录流程中插入日志记录逻辑（major change）
- **logging-infra**: 新建日志基础设施模块，负责日志存储和查询（major change）
- **api-gateway**: 传递 HTTP 请求中的 IP 和 User-Agent 信息（minor change）

## API 变更

### GET /api/admin/login-logs
- 变更类型: 新增
- 描述: 管理员查询登录日志列表，支持时间范围/用户ID过滤，分页
- 兼容性: 新端点

### GET /api/admin/login-logs/:userId
- 变更类型: 新增
- 描述: 查询特定用户的登录历史
- 兼容性: 新端点

### POST /api/auth/login (修改)
- 变更类型: 修改
- 描述: 登录成功后增加异步日志记录
- 兼容性: 向后兼容，响应格式不变

## 数据模型变更

### login_logs (新增)
- 变更类型: 新增表
- 字段: `id` (uuid PK), `user_id` (uuid FK), `login_time` (timestamptz INDEX), `ip_address` (inet/varchar45), `user_agent` (text), `success` (boolean), `created_at` (timestamptz)

## 验收标准

- [ ] 每次用户登录成功时自动记录时间戳、用户ID、IP地址、User-Agent
- [ ] 日志记录异步执行，不增加登录接口响应时间超过 50ms
- [ ] `GET /api/admin/login-logs` 支持按时间范围和用户ID过滤，分页查询
- [ ] `GET /api/admin/login-logs/:userId` 返回指定用户登录历史
- [ ] 日志查询接口需要管理员角色权限
- [ ] 所有查询参数经过类型/格式/范围校验，使用参数化查询
- [ ] ip_address 字段 ≤ 45 字符，user_agent ≤ 512 字符
- [ ] 日志中不包含密码、Token 明文等认证凭证
- [ ] 登录日志记录不可修改或删除（append-only）
- [ ] 管理员查询操作本身记录到审计日志
- [ ] 日志查询每用户每分钟 ≤ 30 次，超限返回 HTTP 429
- [ ] 100万条日志下分页查询响应 ≤ 500ms
- [ ] 日志数据 90 天自动清理
- [ ] 支持用户请求导出/删除自身登录日志数据

## 非功能需求评估

<!-- 由阶段二 NFR 强制评估步骤自动填充 -->
> 根据 `templates/nfr-checklist.md` 强制执行，评估时间: 2026-06-27T07:13:00Z
> 触发: 6/7 维度

| 维度 | 触发 | 评估摘要 |
|------|------|---------|
| 认证/授权 | ✅ | 查询接口需要管理员权限；日志记录仅内部触发 |
| 输入校验 | ✅ | 查询参数校验、SQL注入防护、字段长度约束 |
| 数据保护 | ✅ | 敏感字段不记日志、TLS传输、90天清理、用户数据导出/删除 |
| 速率限制 | ✅ | 每用户每分钟30/60次限流，429响应 |
| 审计日志 | ✅ | 管理员查询操作记录到审计日志；日志不可篡改 |
| 性能 | ✅ | 异步写入≤50ms、查询≤500ms、B-tree/Hash索引、按月分区 |
| 可访问性 | ❌ | 无前端UI变更 |

详见 `requirements/in-progress/REQ-006-impact-analysis.md` 的完整 NFR 评估章节。

## 影响分析

详见 `requirements/in-progress/REQ-006-impact-analysis.md`
- 涉及模块: 3 (auth-service, logging-infra, api-gateway)
- API变更: 3 (2 新增, 1 修改)
- 数据模型变更: 1 新表
- 风险: 3 (日志阻塞登录、数据膨胀、隐私合规)

## 实现跟踪

| 阶段 | 状态 | 完成时间 |
|------|------|---------|
| Design | pending | - |
| Plan | pending | - |
| Build | pending | - |
| Verify | pending | - |
