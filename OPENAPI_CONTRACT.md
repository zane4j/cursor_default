# 论坛系统 API 接口契约初稿（OpenAPI Contract）

## 1. 文档信息

- 文档版本：`v0.1-draft`
- 协议风格：RESTful
- 适用范围：论坛 MVP
- 关联文档：
  - `PRD.md`
  - `ARCHITECTURE.md`

---

## 2. 全局约定

## 2.1 基础信息

- Base URL（dev）：`https://api-dev.example.com`
- Base Path：`/api/v1`
- 数据格式：`application/json; charset=utf-8`
- 时间格式：ISO-8601（UTC），示例：`2026-04-22T10:00:00Z`

## 2.2 认证与鉴权

- 鉴权方案：`Bearer JWT`
- Header：`Authorization: Bearer <access_token>`
- Token 刷新：使用 `refresh_token` 调用刷新接口
- 角色：`USER`、`MODERATOR`、`ADMIN`

## 2.3 请求追踪与幂等

- 每个响应返回 `requestId`
- 写接口支持幂等键（推荐）：
  - Header：`Idempotency-Key: <uuid>`
  - 适用：发帖、评论、举报、审核动作

## 2.4 分页与排序

- 分页参数：
  - `page`：从 1 开始
  - `pageSize`：默认 20，最大 100
- 分页响应：
  - `list`: 数组
  - `page`: 当前页
  - `pageSize`: 页大小
  - `total`: 总条数
- 排序参数（可选）：`sortBy`、`sortOrder`（`asc`/`desc`）

## 2.5 通用响应结构

成功与失败均统一返回：

- `code`：业务码，`0` 表示成功
- `message`：错误或成功提示
- `data`：响应业务数据
- `requestId`：链路追踪 ID

成功示例（语义）：

- `code = 0`
- `message = "OK"`

---

## 3. 错误码约定

| code | HTTP | 说明 |
| --- | --- | --- |
| 0 | 200 | 成功 |
| 40001 | 400 | 参数校验失败 |
| 40101 | 401 | 未登录或 token 无效 |
| 40301 | 403 | 无权限 |
| 40401 | 404 | 资源不存在 |
| 40901 | 409 | 资源冲突（如重复点赞） |
| 42201 | 422 | 业务规则不满足 |
| 42901 | 429 | 请求过于频繁 |
| 50001 | 500 | 服务内部错误 |

常见业务错误补充：

| code | HTTP | 说明 |
| --- | --- | --- |
| 40111 | 401 | 账号被封禁 |
| 40411 | 404 | 帖子不存在或已下架 |
| 40412 | 404 | 评论不存在 |
| 40911 | 409 | 昵称已存在 |
| 40912 | 409 | 已点赞 |
| 40913 | 409 | 已收藏 |

---

## 4. 数据模型（DTO）草案

## 4.1 UserProfileDTO

- `id`: string
- `nickname`: string
- `avatarUrl`: string|null
- `bio`: string|null
- `role`: `USER|MODERATOR|ADMIN`
- `status`: `ACTIVE|BANNED`
- `createdAt`: string

## 4.2 CategoryDTO

- `id`: string
- `name`: string
- `slug`: string
- `description`: string|null
- `postCount`: integer
- `sortOrder`: integer

## 4.3 PostDTO

- `id`: string
- `categoryId`: string
- `title`: string
- `content`: string
- `author`: UserProfileDTO（简化版）
- `status`: `PUBLISHED|HIDDEN|DELETED`
- `viewCount`: integer
- `likeCount`: integer
- `commentCount`: integer
- `favoriteCount`: integer
- `createdAt`: string
- `updatedAt`: string

## 4.4 CommentDTO

- `id`: string
- `postId`: string
- `parentId`: string|null
- `content`: string
- `author`: UserProfileDTO（简化版）
- `status`: `PUBLISHED|DELETED|HIDDEN`
- `createdAt`: string

## 4.5 NotificationDTO

- `id`: string
- `type`: `POST_LIKED|POST_COMMENTED|COMMENT_REPLIED`
- `title`: string
- `content`: string
- `isRead`: boolean
- `relatedPostId`: string|null
- `createdAt`: string

## 4.6 ReportDTO

- `id`: string
- `targetType`: `POST|COMMENT`
- `targetId`: string
- `reasonCode`: string
- `reasonDetail`: string|null
- `status`: `PENDING|RESOLVED|REJECTED`
- `createdAt`: string

## 4.7 MediaUploadTicketDTO

- `objectKey`: string
- `bucket`: string
- `uploadUrl`: string
- `expiresInSeconds`: integer
- `headers`: map<string,string>

---

## 5. API 列表（MVP）

## 5.1 Auth

### POST `/auth/register`

- 说明：邮箱注册
- Auth：否
- Request：
  - `email` string
  - `password` string
  - `nickname` string
- Response `data`：
  - `userId` string

### POST `/auth/login`

- 说明：账号密码登录
- Auth：否
- Request：
  - `email` string
  - `password` string
- Response `data`：
  - `accessToken` string
  - `refreshToken` string
  - `expiresIn` integer
  - `user` UserProfileDTO

### POST `/auth/refresh-token`

- 说明：刷新 token
- Auth：否（使用 refresh token）
- Request：
  - `refreshToken` string
- Response `data`：
  - `accessToken` string
  - `refreshToken` string
  - `expiresIn` integer

### POST `/auth/logout`

- 说明：退出登录，失效 refresh token
- Auth：是（USER+）
- Request：空
- Response：`data = null`

---

## 5.2 User

### GET `/users/me`

- 说明：获取当前用户资料
- Auth：是（USER+）
- Response `data`：UserProfileDTO

### PATCH `/users/me`

- 说明：修改当前用户资料
- Auth：是（USER+）
- Request（至少一个字段）：
  - `nickname` string（2-24）
  - `avatarUrl` string
  - `bio` string
- Response `data`：UserProfileDTO
- 可能错误：`40911`（昵称冲突）

---

## 5.3 Category

### GET `/categories`

- 说明：获取板块列表
- Auth：否
- Query（可选）：
  - `includeHidden` boolean（仅管理员可用）
- Response `data`：
  - `list` CategoryDTO[]

---

## 5.4 Post

### GET `/posts`

- 说明：帖子列表
- Auth：否
- Query：
  - `categoryId` string（可选）
  - `keyword` string（可选）
  - `sortBy` enum: `createdAt|hotScore`
  - `sortOrder` enum: `asc|desc`
  - `page` int
  - `pageSize` int
- Response `data`：
  - 分页对象，`list` 为 PostDTO[]

### GET `/posts/{postId}`

- 说明：帖子详情
- Auth：否（下架内容仅管理员可见）
- Path：
  - `postId` string
- Response `data`：PostDTO（详情版）
- 可能错误：`40411`

### POST `/posts`

- 说明：创建帖子
- Auth：是（USER+）
- Headers：可选 `Idempotency-Key`
- Request：
  - `categoryId` string
  - `title` string（5-100）
  - `content` string
  - `attachments` string[]（可选，object key 列表）
- Response `data`：
  - `postId` string

### PATCH `/posts/{postId}`

- 说明：编辑帖子（作者/管理员）
- Auth：是（USER+）
- Path：
  - `postId` string
- Request：
  - `title` string（可选）
  - `content` string（可选）
- Response `data`：PostDTO

### DELETE `/posts/{postId}`

- 说明：删除帖子（软删除）
- Auth：是（作者/管理员）
- Path：
  - `postId` string
- Response：`data = null`

### POST `/posts/{postId}/view`

- 说明：帖子浏览计数（防刷由服务端处理）
- Auth：否
- Response：`data = null`

---

## 5.5 Comment

### GET `/posts/{postId}/comments`

- 说明：评论列表
- Auth：否
- Query：
  - `page` int
  - `pageSize` int
- Response `data`：
  - 分页对象，`list` 为 CommentDTO[]

### POST `/posts/{postId}/comments`

- 说明：发表评论
- Auth：是（USER+）
- Headers：可选 `Idempotency-Key`
- Request：
  - `content` string
  - `parentId` string（可选）
- Response `data`：
  - `commentId` string

### DELETE `/comments/{commentId}`

- 说明：删除评论（作者/管理员）
- Auth：是（USER+）
- Path：
  - `commentId` string
- Response：`data = null`

---

## 5.6 Interaction

### POST `/posts/{postId}/like`

- 说明：点赞帖子
- Auth：是（USER+）
- Response：`data = null`
- 可能错误：`40912`

### DELETE `/posts/{postId}/like`

- 说明：取消点赞
- Auth：是（USER+）
- Response：`data = null`

### POST `/posts/{postId}/favorite`

- 说明：收藏帖子
- Auth：是（USER+）
- Response：`data = null`
- 可能错误：`40913`

### DELETE `/posts/{postId}/favorite`

- 说明：取消收藏
- Auth：是（USER+）
- Response：`data = null`

### GET `/users/me/favorites`

- 说明：我的收藏列表
- Auth：是（USER+）
- Query：`page`、`pageSize`
- Response `data`：分页 PostDTO[]

---

## 5.7 Notification

### GET `/users/me/notifications`

- 说明：通知列表
- Auth：是（USER+）
- Query：
  - `isRead` boolean（可选）
  - `page` int
  - `pageSize` int
- Response `data`：分页 NotificationDTO[]

### PATCH `/users/me/notifications/{notificationId}/read`

- 说明：单条通知标记已读
- Auth：是（USER+）
- Response：`data = null`

### PATCH `/users/me/notifications/read-all`

- 说明：全部通知标记已读
- Auth：是（USER+）
- Response：`data = null`

---

## 5.8 Report（用户举报）

### POST `/reports`

- 说明：提交举报
- Auth：是（USER+）
- Headers：可选 `Idempotency-Key`
- Request：
  - `targetType` enum: `POST|COMMENT`
  - `targetId` string
  - `reasonCode` string
  - `reasonDetail` string（可选）
- Response `data`：
  - `reportId` string

### GET `/users/me/reports`

- 说明：我的举报记录
- Auth：是（USER+）
- Query：`page`、`pageSize`
- Response `data`：分页 ReportDTO[]

---

## 5.9 Moderation（版主/管理员）

### GET `/moderation/reports`

- 说明：举报列表
- Auth：是（MODERATOR+）
- Query：
  - `status` enum: `PENDING|RESOLVED|REJECTED`（可选）
  - `targetType` enum: `POST|COMMENT`（可选）
  - `page` int
  - `pageSize` int
- Response `data`：分页 ReportDTO[]

### PATCH `/moderation/reports/{reportId}/resolve`

- 说明：处理举报
- Auth：是（MODERATOR+）
- Headers：可选 `Idempotency-Key`
- Request：
  - `action` enum: `IGNORE|HIDE_CONTENT|DELETE_CONTENT|BAN_USER`
  - `comment` string（可选）
- Response：`data = null`

### PATCH `/moderation/posts/{postId}/status`

- 说明：变更帖子状态
- Auth：是（MODERATOR+）
- Request：
  - `status` enum: `PUBLISHED|HIDDEN|DELETED`
  - `reason` string（可选）
- Response：`data = null`

### PATCH `/moderation/users/{userId}/status`

- 说明：封禁/解封用户
- Auth：是（ADMIN）
- Request：
  - `status` enum: `ACTIVE|BANNED`
  - `reason` string（可选）
- Response：`data = null`

---

## 5.10 Media（MinIO）

### POST `/media/upload-ticket`

- 说明：获取 MinIO 预签名上传凭证
- Auth：是（USER+）
- Request：
  - `bizType` enum: `AVATAR|POST_IMAGE|ATTACHMENT`
  - `fileName` string
  - `contentType` string
  - `fileSize` integer
- Response `data`：MediaUploadTicketDTO

### POST `/media/confirm`

- 说明：上传完成后回调，登记资源元数据
- Auth：是（USER+）
- Request：
  - `objectKey` string
  - `etag` string（可选）
  - `fileSize` integer
  - `sha256` string（可选）
  - `bizRefId` string（可选）
- Response `data`：
  - `mediaId` string
  - `accessUrl` string（短时效）

### GET `/media/{mediaId}/download-url`

- 说明：获取下载链接（时效 URL）
- Auth：是（具备目标资源访问权限）
- Query：
  - `expiresIn` int（可选，默认 300，最大 1800）
- Response `data`：
  - `downloadUrl` string
  - `expiresAt` string

---

## 6. 限流与安全策略（接口层）

建议默认限流（按用户 ID + IP 双维度）：

- 登录：`10 次/分钟`
- 发帖：`5 次/分钟`
- 评论：`20 次/分钟`
- 举报：`10 次/小时`
- 上传凭证申请：`30 次/小时`

安全校验要求：

1. DTO 参数校验必须开启（长度、枚举、格式）
2. 写接口必须鉴权 + 权限校验
3. 管理类接口必须写审计日志
4. 富文本内容需服务端安全清洗

---

## 7. 版本管理与兼容策略

1. API 主版本在路径中体现：`/api/v1`
2. 非兼容改动必须升级版本（如 `v2`）
3. 字段新增遵循向后兼容原则：
   - 尽量新增可选字段
   - 不删除已发布字段（先标记废弃）
4. 废弃策略：
   - 文档标记 `deprecated`
   - 至少保留一个迭代周期

---

## 8. 待确认项（接口评审）

1. 注册是否支持手机号与验证码登录并行
2. 热门排序 `hotScore` 的计算口径
3. 评论是否立即支持多级回复
4. 举报原因枚举是否由运营后台可配置
5. 媒体下载 URL 最长有效期策略

