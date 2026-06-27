---
req_id: REQ-006
req_title: 添加用户登录日志功能
analysis_date: "2026-06-27T07:12:00Z"
analyzed_by: AI
kb_contexts_used:
  - knowledge-base/architecture/overview.md
confidence: low
---

# 影响分析: 添加用户登录日志功能

> ⚠️ **置信度: low** — 当前 KB 无匹配的认证/日志模块。此影响分析基于通用软件工程推理，非基于现有代码分析。随 KB 扩展应重新评估。

## 涉及模块

### auth-service (新建或扩展)
- **影响程度**: major
- **变更描述**: 在现有认证流程（或新建认证模块）中插入登录日志记录逻辑。登录成功后触发日志事件。
- **预计修改文件数**: 3-5（login handler, auth middleware, event emitter/hook）

### logging-infra (新建)
- **影响程度**: major
- **变更描述**: 新建日志基础设施模块，负责登录日志的持久化存储、查询接口。需要数据模型定义和 API 端点。
- **预计修改文件数**: 4-6（log model, log service, log API routes, migrations）

### api-gateway (可能受影响)
- **影响程度**: minor
- **变更描述**: 如果需要从 HTTP 请求中提取 IP 和 User-Agent，API gateway 层需传递这些信息到下游认证服务。
- **预计修改文件数**: 1-2

## API 变更

### GET /api/admin/login-logs
- **变更类型**: 新增
- **描述**: 管理员查询登录日志列表，支持按时间范围/用户/IP 过滤
- **兼容性**: 新端点，不影响现有 API

### GET /api/admin/login-logs/:userId
- **变更类型**: 新增
- **描述**: 查询特定用户的登录历史
- **兼容性**: 新端点，不影响现有 API

### POST /api/auth/login (修改)
- **变更类型**: 修改
- **描述**: 登录成功后增加日志记录副作用（异步，不阻塞登录响应）
- **兼容性**: 响应格式不变，向后兼容

## 数据模型变更

### login_logs (新增)
- **变更类型**: 新增表
- **字段变更**:
  - `id` (uuid, PK)
  - `user_id` (uuid, FK → users)
  - `login_time` (timestamp with timezone, NOT NULL, INDEX)
  - `ip_address` (inet or varchar(45), NOT NULL)
  - `user_agent` (text, nullable)
  - `success` (boolean, default true)
  - `created_at` (timestamp with timezone)
- **迁移需求**: 新建 migration，无数据迁移（新表）

## 风险评估

- **风险**: 登录日志写入失败可能阻塞登录流程
- **严重度**: high
- **缓解措施**: 日志写入使用异步队列/后台任务，与登录主流程解耦。日志写入失败仅记录错误但不影响登录返回

- **风险**: 登录日志数据量随用户基数增长可能快速膨胀
- **严重度**: medium
- **缓解措施**: 设置日志保留策略（如 90 天自动清理），对 `login_time` 建立分区表

- **风险**: IP 地址和 User-Agent 属于用户行为数据，涉及隐私合规
- **严重度**: medium
- **缓解措施**: 在隐私政策中披露日志收集，提供数据导出和删除能力。详见 NFR 数据保护评估

---

## 非功能需求评估

> 根据 `templates/nfr-checklist.md` 强制执行，评估时间: 2026-06-27T07:13:00Z
> 触发: 6/7 维度，未触发: 1/7

### 1. 认证/授权
- 触发: ✅ (`type=feature` + 涉及用户登录操作)
- 评估: 登录日志查询接口需要管理员权限，日志记录接口由认证模块内部调用（登录成功后触发）
- 补充验收标准:
  - [ ] `GET /api/admin/login-logs` 需要管理员角色权限（Role: admin）
  - [ ] `GET /api/admin/login-logs/:userId` 需要管理员角色或本用户权限
  - [ ] 日志记录端点不接受外部直接调用，仅由 auth 模块内部触发

### 2. 输入校验
- 触发: ✅ (涉及 API 变更 + 数据模型变更)
- 评估: 新 API 端点需要参数校验；新数据模型字段需要类型和长度约束
- 补充验收标准:
  - [ ] `GET /api/admin/login-logs` 的查询参数（`start_date`, `end_date`）必须校验为合法 ISO 8601 日期格式
  - [ ] `userId` 路径参数必须校验为合法 UUID 格式
  - [ ] 分页参数 `limit` 上限 ≤ 100，`offset` ≥ 0，防止全表扫描
  - [ ] `ip_address` 字段长度 ≤ 45 字符（IPv6 最大长度）
  - [ ] `user_agent` 字段长度 ≤ 512 字符，防止超长字符串注入
  - [ ] 所有输入参数需要 SQL 注入防护（使用参数化查询/ORM）

### 3. 数据保护
- 触发: ✅ (`type=feature` + 涉及用户数据: IP/User-Agent/时间戳)
- 评估: IP 地址和 User-Agent 属于用户行为数据，需遵循最小收集、安全存储、日志脱敏原则
- 补充验收标准:
  - [ ] 数据库连接使用 TLS 加密传输
  - [ ] 日志中禁止记录用户密码、Token 明文、Session ID 等认证凭证
  - [ ] IP 地址和 User-Agent 的日志输出仅保留前 8 字符哈希（用于调试但不暴露完整值）
  - [ ] 提供数据保留策略：登录日志默认保留 90 天，超期自动清理
  - [ ] 支持用户请求导出和删除其自身的登录日志数据（GDPR/隐私合规）

### 4. 速率限制
- 触发: ✅ (新增 API 端点: `GET /api/admin/login-logs*`)
- 评估: 管理员查询接口存在被滥用的风险（高频查询消耗数据库资源）
- 补充验收标准:
  - [ ] `GET /api/admin/login-logs` 每用户每分钟 ≤ 30 次请求
  - [ ] `GET /api/admin/login-logs/:userId` 每用户每分钟 ≤ 60 次请求
  - [ ] 超过限流阈值返回 HTTP 429 并包含 `Retry-After` 头

### 5. 审计日志
- 触发: ✅ (`priority=medium` + 涉及认证操作)
- 评估: 登录日志功能本身就是审计工具。但需要确保日志记录的完整性和防篡改
- 补充验收标准:
  - [ ] 登录日志记录本身的操作（管理员查询/导出）需要记录到独立的审计日志中
  - [ ] 审计日志条目包含：操作时间、操作者ID、操作类型（query/export）、目标资源
  - [ ] 登录日志记录不可被普通管理员修改或删除（immutable/append-only）
  - [ ] 日志中禁止包含：密码、Token 明文、完整 Session ID

### 6. 性能
- 触发: ✅ (`type=feature` + 涉及数据查询)
- 评估: 登录日志表随用户基数增长，高频写入+查询需要索引策略和分区设计
- 补充验收标准:
  - [ ] 登录日志写入操作响应时间 ≤ 50ms（异步，不阻塞登录主流程）
  - [ ] `GET /api/admin/login-logs` 在 100 万条日志数据下，默认分页查询响应 ≤ 500ms
  - [ ] `login_time` 字段建立 B-tree 索引，支持范围查询
  - [ ] `user_id` 字段建立 Hash 索引，支持用户维度查询
  - [ ] 日志表按月分区，自动清理过期分区（90 天保留策略）

### 7. 可访问性
- 触发: ❌ (无前端 UI 变更)
- 评估: 本需求仅涉及后端 API 和数据库变更，不涉及前端 UI。如后续增加管理界面，需重新评估此维度。
