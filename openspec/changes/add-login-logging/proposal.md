## Why

当前系统缺少登录行为的审计追踪能力。安全团队无法获知谁在何时从何处登录，这在安全事件调查和合规审计中是一个关键缺口。添加登录日志功能可以记录每次登录的时间、来源IP和User-Agent信息，为安全审计和异常登录检测提供数据基础。

## What Changes

- **新增** `login_logs` 数据表，存储登录时间戳、用户ID、IP地址、User-Agent等信息
- **新增** `GET /api/admin/login-logs` 管理端API，支持按时间范围、用户等多条件查询登录日志
- **新增** `GET /api/admin/login-logs/:userId` 管理端API，查询特定用户的登录历史
- **修改** `POST /api/auth/login` 登录接口，在登录成功后异步记录日志（不阻塞登录响应）
- 日志记录采用异步队列机制，与登录主流程解耦
- 日志数据保留90天自动清理策略

## Capabilities

### New Capabilities
- `login-logging`: 用户登录行为日志记录与查询功能。包括登录事件自动采集（时间戳/IP/UA）、管理端日志查询API、日志数据保留策略。安全要求：日志记录不可篡改（append-only），管理员查询操作本身记入审计日志。

### Modified Capabilities
<!-- 无现有 capability 的 spec 级需求变更 -->

## Impact

- **代码**: `auth-service` 登录 handler 增加日志记录逻辑；新建 `logging-infra` 模块（log model + service + API routes + migration）
- **API**: 新增 2 个管理端 GET 端点，修改 1 个现有登录 POST 端点（向后兼容）
- **数据库**: 新增 `login_logs` 表（含索引和分区策略），需要执行 migration
- **依赖**: 引入异步任务队列/消息机制用于日志写入解耦
- **基础设施**: 需要按月分区存储策略和90天自动清理 Job
