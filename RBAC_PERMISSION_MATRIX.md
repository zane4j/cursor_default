# 论坛系统 RBAC 权限矩阵（角色-资源-动作）

## 1. 文档信息

- 文档版本：`v1.0-draft`
- 适用范围：论坛 MVP（前台 + 管理后台）
- 关联文档：
  - `PRD.md`
  - `OPENAPI_CONTRACT.md`
  - `ERD_DDL.md`
  - `MODERATION_WORKFLOW.md`

---

## 2. 目标与设计原则

## 2.1 目标

1. 将“谁可以对什么资源做什么动作”一次性固化，减少权限返工。
2. 为后端鉴权（Spring Security）提供可执行策略。
3. 为前端权限显隐提供统一依据，避免“前端可见但后端拒绝”。

## 2.2 原则

1. **默认拒绝**：未显式授权即拒绝。
2. **最小权限**：角色仅拥有完成职责所需权限。
3. **纵深防御**：前端显隐 + 后端强校验双重控制。
4. **可审计**：管理动作必须落审计日志。

---

## 3. 角色定义

1. `GUEST`：未登录访客。
2. `USER`：普通注册用户。
3. `MODERATOR`：版主（可管理指定板块）。
4. `ADMIN`：管理员（全站治理与配置）。

说明：

- `MODERATOR` 默认只对“授权板块”生效，超出板块范围等同普通用户。
- `ADMIN` 默认具备全站范围权限。

---

## 4. 资源与动作定义

## 4.1 资源（Resource）

1. `AUTH`：认证（注册、登录、刷新、登出）
2. `PROFILE`：个人资料与账号设置
3. `CATEGORY`：板块
4. `POST`：帖子
5. `COMMENT`：评论
6. `INTERACTION`：点赞/收藏
7. `NOTIFICATION`：通知
8. `REPORT`：举报提交与个人举报记录
9. `MODERATION`：举报处理、内容处置、封禁
10. `MEDIA`：上传凭证、媒体确认、下载链接
11. `AUDIT_LOG`：审计日志

## 4.2 动作（Action）

基础动作：

- `CREATE`
- `READ`
- `UPDATE`
- `DELETE`
- `LIST`

治理动作：

- `HANDLE_REPORT`
- `HIDE_CONTENT`
- `DELETE_CONTENT`
- `BAN_USER`
- `UNBAN_USER`

辅助动作：

- `MARK_READ`
- `ISSUE_UPLOAD_TICKET`
- `CONFIRM_UPLOAD`

---

## 5. 角色-资源-动作矩阵

标记说明：

- `Y`：允许
- `N`：不允许
- `C`：条件允许（见“条件规则”）

| 资源/动作 | GUEST | USER | MODERATOR | ADMIN |
| --- | --- | --- | --- | --- |
| AUTH.CREATE（注册） | Y | N | N | N |
| AUTH.READ（登录态查询） | N | Y | Y | Y |
| AUTH.UPDATE（刷新 token） | N | Y | Y | Y |
| AUTH.DELETE（登出） | N | Y | Y | Y |
| PROFILE.READ（查看自己） | N | Y | Y | Y |
| PROFILE.UPDATE（修改自己） | N | Y | Y | Y |
| CATEGORY.LIST/READ | Y | Y | Y | Y |
| POST.LIST/READ（公开内容） | Y | Y | Y | Y |
| POST.CREATE | N | Y | Y | Y |
| POST.UPDATE | N | C | C | Y |
| POST.DELETE（软删） | N | C | C | Y |
| COMMENT.LIST/READ | Y | Y | Y | Y |
| COMMENT.CREATE | N | Y | Y | Y |
| COMMENT.DELETE | N | C | C | Y |
| INTERACTION.CREATE（点赞/收藏） | N | Y | Y | Y |
| INTERACTION.DELETE（取消） | N | Y | Y | Y |
| NOTIFICATION.LIST/READ | N | Y | Y | Y |
| NOTIFICATION.MARK_READ | N | Y | Y | Y |
| REPORT.CREATE | N | Y | Y | Y |
| REPORT.LIST（我的举报） | N | Y | Y | Y |
| MODERATION.HANDLE_REPORT | N | N | C | Y |
| MODERATION.HIDE_CONTENT | N | N | C | Y |
| MODERATION.DELETE_CONTENT | N | N | C | Y |
| MODERATION.BAN_USER | N | N | N | Y |
| MODERATION.UNBAN_USER | N | N | N | Y |
| MEDIA.ISSUE_UPLOAD_TICKET | N | Y | Y | Y |
| MEDIA.CONFIRM_UPLOAD | N | Y | Y | Y |
| MEDIA.READ_DOWNLOAD_URL | N | C | C | C |
| AUDIT_LOG.LIST/READ | N | N | C | Y |

---

## 6. 条件规则（C）

## 6.1 所有权规则（Owner）

1. `POST.UPDATE/DELETE`：仅帖子作者可操作，`ADMIN` 例外。
2. `COMMENT.DELETE`：仅评论作者可操作，`ADMIN` 例外。
3. `MEDIA.READ_DOWNLOAD_URL`：需具备关联业务对象访问权限。

## 6.2 版主范围规则（Moderator Scope）

1. `MODERATOR` 的治理动作仅作用于“被授权板块”。
2. 跨板块目标资源，`MODERATOR` 无权操作。
3. 版主范围关系建议持久化表：`moderator_category_scopes`（后续可加）。

## 6.3 账号状态规则（User Status）

1. `BANNED` 用户可限制为：
   - 禁止登录，或
   - 可登录但禁止写操作（推荐明确一种并在系统全局统一）
2. `BANNED` 用户不允许 `POST.CREATE`、`COMMENT.CREATE`、`REPORT.CREATE`、`INTERACTION.CREATE`。

## 6.4 内容状态规则（Post/Comment Status）

1. `HIDDEN/DELETED` 内容对 `GUEST` 和 `USER` 默认不可见。
2. `MODERATOR/ADMIN` 可按权限查看处置内容（用于审核追溯）。

---

## 7. API 与权限映射（按接口）

## 7.1 Auth

- `POST /auth/register` -> `GUEST`
- `POST /auth/login` -> `GUEST`
- `POST /auth/refresh-token` -> 持有 refresh token
- `POST /auth/logout` -> `USER+`

## 7.2 User/Profile

- `GET /users/me` -> `USER+`
- `PATCH /users/me` -> `USER+`

## 7.3 Category/Post/Comment

- `GET /categories` -> 全角色
- `GET /posts`、`GET /posts/{id}` -> 全角色（隐藏帖受限）
- `POST /posts` -> `USER+` 且未封禁
- `PATCH /posts/{id}`、`DELETE /posts/{id}` -> 作者或 `ADMIN`（`MODERATOR` 按板块规则可处置）
- `POST /posts/{id}/comments` -> `USER+`
- `DELETE /comments/{id}` -> 评论作者或 `ADMIN`

## 7.4 Interaction/Notification/Report

- `POST|DELETE /posts/{id}/like` -> `USER+`
- `POST|DELETE /posts/{id}/favorite` -> `USER+`
- `GET /users/me/notifications`、`PATCH .../read` -> `USER+`
- `POST /reports`、`GET /users/me/reports` -> `USER+`

## 7.5 Moderation/Admin

- `GET /moderation/reports` -> `MODERATOR+`
- `PATCH /moderation/reports/{id}/resolve` -> `MODERATOR+`（作用域校验）
- `PATCH /moderation/posts/{id}/status` -> `MODERATOR+`（作用域校验）
- `PATCH /moderation/users/{id}/status` -> `ADMIN`

## 7.6 Media

- `POST /media/upload-ticket` -> `USER+`
- `POST /media/confirm` -> `USER+`
- `GET /media/{id}/download-url` -> 资源访问权限校验通过

---

## 8. Spring Boot 3 落地建议

## 8.1 权限表达建议

在代码中使用“角色 + 细粒度权限”组合：

1. 角色：`ROLE_USER`、`ROLE_MODERATOR`、`ROLE_ADMIN`
2. 权限：`post:write`、`moderation:handle`、`user:ban` 等

建议表达式：

- `@PreAuthorize("hasRole('ADMIN') or @authz.isPostOwner(#postId)")`
- `@PreAuthorize("hasRole('ADMIN') or @authz.isModeratorInScope(#postId)")`

## 8.2 校验顺序

1. 认证校验（是否登录）
2. 角色校验（是否具备基础角色）
3. 资源级校验（所有权、板块范围、状态）
4. 业务规则校验（封禁、频率、幂等）

## 8.3 审计要求

以下动作必须记录到 `audit_logs`：

1. 举报处理
2. 内容下架/删除
3. 用户封禁/解封
4. 管理员修改高风险配置（后续）

---

## 9. 前端显隐规则（与后端一致）

1. 未登录：
   - 隐藏点赞/收藏/评论输入/发帖入口。
   - 点击受限操作时弹登录引导。
2. 已登录普通用户：
   - 仅展示“编辑/删除本人内容”按钮。
3. 版主：
   - 在授权板块显示“审核处理”入口。
4. 管理员：
   - 显示全站治理入口（举报中心、用户状态管理）。

注意：前端显隐仅提升体验，不能代替后端鉴权。

---

## 10. 验收清单

- [ ] 所有受限接口均配置鉴权注解/策略
- [ ] 所有 `C` 权限均实现条件校验（owner/scope/status）
- [ ] 管理动作审计日志完整（包含 requestId）
- [ ] 封禁用户写操作被统一拒绝
- [ ] 前端显隐与后端权限无冲突
- [ ] 权限拒绝返回统一错误码 `40301`

---

## 11. 待确认事项

1. `MODERATOR` 是否允许直接删除内容，或仅允许“下架 + 提交管理员复核”。
2. 封禁策略是“禁止登录”还是“可登录但禁言”。
3. 是否需要新增只读运营角色（如 `AUDITOR`）。
4. 是否将“下载私有附件”限制为发帖人/作者本人。

