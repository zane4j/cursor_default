# 论坛系统举报与审核流程（状态机规范）

## 1. 文档信息

- 文档版本：`v1.0-draft`
- 适用范围：论坛 MVP（帖子/评论举报、版主/管理员审核）
- 关联文档：
  - `PRD.md`
  - `OPENAPI_CONTRACT.md`
  - `ERD_DDL.md`
  - `RBAC_PERMISSION_MATRIX.md`

---

## 2. 目标与边界

## 2.1 目标

1. 固化举报到审核闭环，减少“规则不一致”导致的运营风险。
2. 定义明确的状态机和动作语义，确保研发、测试、运营一致理解。
3. 保证每一次审核动作可追踪、可审计、可复盘。

## 2.2 边界（MVP）

本规范覆盖：

1. 举报对象：`POST`、`COMMENT`
2. 处理角色：`MODERATOR`、`ADMIN`
3. 处理动作：忽略、下架、删除、封禁
4. 审计与通知：审核日志、结果通知（可选）

本期不覆盖：

1. AI 自动审核判定
2. 跨站联合风控
3. 申诉工单系统（仅预留状态）

---

## 3. 术语定义

1. **举报（Report）**：用户对帖子或评论发起的违规反馈。
2. **审核单（Case）**：运营处理单元，可由一条或多条举报聚合（MVP 先 1:1）。
3. **目标内容（Target）**：被举报对象，类型为帖子或评论。
4. **处理动作（Action）**：审核员执行的业务动作，改变举报状态及目标内容状态。
5. **审计日志（Audit Log）**：记录操作人、动作、前后状态、理由与时间。

---

## 4. 举报状态机（Report）

## 4.1 状态定义

举报状态采用：

- `PENDING`：待处理
- `RESOLVED`：已处理并生效
- `REJECTED`：已驳回（认定无违规）
- `CANCELLED`：已撤销（系统或用户撤销，MVP 可后置实现）

## 4.2 状态迁移

允许迁移：

1. `PENDING -> RESOLVED`
2. `PENDING -> REJECTED`
3. `PENDING -> CANCELLED`（可选）

禁止迁移（MVP）：

1. `RESOLVED -> PENDING`
2. `REJECTED -> PENDING`
3. `CANCELLED -> PENDING`

如需“重开”，建议新建举报单，避免历史语义混乱。

## 4.3 触发条件

1. 用户提交举报：创建 `PENDING`
2. 版主/管理员处理并生效：进入 `RESOLVED`
3. 版主/管理员判定无效举报：进入 `REJECTED`
4. 系统判定无效（如目标已永久删除且超窗口期）：可置 `CANCELLED`

---

## 5. 审核动作状态机（Action -> Target）

## 5.1 动作字典

| action | 语义 | 适用对象 | 最小角色 |
| --- | --- | --- | --- |
| `IGNORE` | 忽略举报，不改目标状态 | POST/COMMENT | MODERATOR |
| `HIDE_CONTENT` | 下架内容，对普通用户不可见 | POST/COMMENT | MODERATOR |
| `DELETE_CONTENT` | 逻辑删除内容 | POST/COMMENT | MODERATOR |
| `BAN_USER` | 封禁目标作者账号 | POST/COMMENT | ADMIN |

## 5.2 动作与结果映射

| action | report.status | target.status | user.status |
| --- | --- | --- | --- |
| `IGNORE` | `REJECTED` | 不变 | 不变 |
| `HIDE_CONTENT` | `RESOLVED` | `HIDDEN` | 不变 |
| `DELETE_CONTENT` | `RESOLVED` | `DELETED` 或 `deleted_at!=null` | 不变 |
| `BAN_USER` | `RESOLVED` | 通常配合 `HIDDEN/DELETED` | `BANNED` |

说明：

1. `BAN_USER` 建议总是与内容动作联动（隐藏或删除），避免封禁后违规内容仍可见。
2. 如果目标已是 `DELETED`，`DELETE_CONTENT` 视为幂等成功。

---

## 6. 业务流程（时序）

## 6.1 用户举报流程

1. 用户选择举报对象（帖子/评论）并填写原因分类。
2. 系统校验：
   - 用户登录状态
   - 举报频率限制
   - 目标存在性与可举报性
3. 创建 `reports` 记录，状态为 `PENDING`。
4. 返回举报受理成功。

## 6.2 版主审核流程

1. 审核员进入举报列表，按状态筛选 `PENDING`。
2. 查看举报详情（目标内容、历史举报、作者信息）。
3. 选择动作并填写处理备注。
4. 系统执行事务：
   - 更新 `report.status`
   - 更新目标内容状态（如需）
   - 更新用户状态（如 `BAN_USER`）
   - 写入 `audit_logs`
5. 返回处理结果并刷新列表。

## 6.3 处理后通知流程（建议）

1. 对举报人发送“已处理”通知（可选）。
2. 对被处理作者发送“内容处理结果”通知（建议）。
3. 失败不影响主事务（异步通知）。

---

## 7. 并发与一致性策略

## 7.1 重复处理保护

1. 审核接口要求 `Idempotency-Key`（推荐）。
2. 处理前校验 `report.status == PENDING`。
3. 若非 `PENDING` 返回业务冲突（如 `409`），并提示已被处理。

## 7.2 事务边界

单次审核动作应在一个事务内完成：

1. 更新举报状态
2. 更新目标状态
3. 更新用户状态（如需要）
4. 写审计日志

任一步骤失败则回滚，避免“半处理”。

## 7.3 锁策略（建议）

1. 对 `reports.id` 使用行级锁（`for update`）防并发处理。
2. 内容状态更新使用条件更新（基于当前状态）保证幂等。

---

## 8. 审计与追踪规范

## 8.1 必记字段

每次审核动作必须记录：

1. `operator_user_id`
2. `action`
3. `target_type`
4. `target_id`
5. `before_data`
6. `after_data`
7. `request_id`
8. `created_at`

## 8.2 审计查询能力

后台至少支持：

1. 按操作人查询
2. 按目标内容查询
3. 按时间范围查询
4. 按动作类型筛选

---

## 9. API 对齐（与 OPENAPI_CONTRACT）

核心接口映射：

1. 提交举报：`POST /api/v1/reports`
2. 举报列表：`GET /api/v1/moderation/reports`
3. 处理举报：`PATCH /api/v1/moderation/reports/{reportId}/resolve`
4. 帖子状态变更：`PATCH /api/v1/moderation/posts/{postId}/status`
5. 用户状态变更：`PATCH /api/v1/moderation/users/{userId}/status`

返回规范：

1. 统一响应体（`code/message/data/requestId`）
2. 状态冲突返回 `409`（例如举报已处理）
3. 权限不足返回 `403`

---

## 10. 可观测性指标（审核域）

建议监控指标：

1. `report_pending_count`：待处理举报数
2. `report_handle_latency_p95`：举报处理时延 P95
3. `report_resolved_rate`：处理率
4. `report_rejected_rate`：驳回率
5. `moderation_action_error_rate`：审核动作失败率
6. `ban_user_count_daily`：每日封禁人数

告警建议：

1. `PENDING` 超过阈值（如 > 500）告警
2. 处理时延超 SLA（如 24h）告警
3. 审核接口错误率 > 2% 告警

---

## 11. SLA 与运营规则建议

1. 普通举报处理 SLA：24 小时内
2. 高危举报（涉政/涉暴等）：2 小时内（可通过 reasonCode 标记）
3. 审核员必须填写处理备注（最少 5 字）
4. 封禁操作默认需要管理员权限与二次确认

---

## 12. 测试用例建议（Given/When/Then）

1. Given 举报为 `PENDING`，When 版主执行 `HIDE_CONTENT`，Then 举报状态变 `RESOLVED` 且目标状态为 `HIDDEN`。
2. Given 举报为 `PENDING`，When 版主执行 `IGNORE`，Then 举报状态变 `REJECTED` 且目标状态不变。
3. Given 举报已 `RESOLVED`，When 再次处理，Then 返回冲突错误并不产生二次审计。
4. Given 普通用户调用审核接口，When 请求处理，Then 返回 `403`。
5. Given 管理员执行 `BAN_USER`，When 操作成功，Then 用户状态变 `BANNED` 且写入审计日志。

---

## 13. 与数据模型映射

1. 状态源：`reports.status`
2. 目标状态：`posts.status` / `comments.status`
3. 用户状态：`users.status`
4. 审计记录：`audit_logs`

建议补充字段：

1. `reports.priority`（可选）
2. `reports.source`（用户端/系统端）
3. `reports.version`（乐观锁，可选）

---

## 14. 待确认事项

1. 是否启用 `CANCELLED` 状态（用户撤销举报）。
2. `BAN_USER` 是否允许版主执行（当前建议仅 ADMIN）。
3. 是否要求“封禁必须联动下架全部历史内容”。
4. 举报处理通知是否默认开启。
5. 是否在 MVP 阶段支持举报聚合（多举报并单）。

