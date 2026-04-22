# 论坛系统架构设计文档（Java 技术栈版）

## 1. 文档目标与范围

本文档用于指导从 0 到 1 构建一个支持 Web（PC）与移动端浏览器（H5）的论坛系统，覆盖：

- 业务能力边界（MVP 与演进路线）
- 技术选型与架构分层
- 核心模块与数据模型
- 接口规范与安全策略
- 部署、监控、运维与扩展方案

后端技术栈固定为：

- `JDK 21`
- `Spring Boot 3.x`
- `PostgreSQL`
- `Redis`
- `MinIO`（自建对象存储）

---

## 2. 产品目标与非目标

### 2.1 MVP 目标

1. 用户注册、登录、鉴权、基础个人资料
2. 板块管理与帖子发布/编辑/删除
3. 评论回复（含楼中楼扩展能力）
4. 点赞、收藏、通知
5. 基础审核（举报、删帖、封禁）
6. 响应式前端，支持手机与 PC

### 2.2 非目标（第一阶段不做）

1. 微服务拆分（优先单体模块化）
2. 复杂推荐系统
3. 即时聊天
4. 多租户能力

---

## 3. 总体架构

采用“前后端分离 + 模块化单体”架构，先保证开发效率与可维护性，再按流量增长平滑演进。

### 3.1 架构图（逻辑）

Client（Mobile/PC Browser）
-> CDN/Nginx
-> Frontend（Next.js/React）
-> API Gateway（Nginx 路由）
-> Forum Backend（Spring Boot 3, JDK 21）
-> PostgreSQL / Redis / MinIO

### 3.2 分层原则

后端采用经典分层并结合领域模块隔离：

1. `controller`：HTTP 接口层（参数校验、DTO）
2. `application`：应用服务层（用例编排、事务边界）
3. `domain`：领域模型层（实体、值对象、领域规则）
4. `infrastructure`：基础设施层（JPA/MyBatis、Redis、MinIO、MQ）

这样可以在当前单体架构中保持清晰边界，为后续服务拆分保留路径。

---

## 4. 前端架构（移动端 + PC）

### 4.1 技术建议

- `Next.js + TypeScript`
- `Tailwind CSS` 或 `Ant Design + 响应式栅格`
- 数据请求：`TanStack Query`
- 表单：`React Hook Form + Zod`

### 4.2 响应式策略

1. Mobile First：先实现手机端，再扩展大屏
2. 断点建议：`<768`（手机）、`768-1023`（平板）、`>=1024`（PC）
3. 布局策略：
   - 帖子列表：手机单列，PC 双栏
   - 发帖编辑：手机全宽，PC 居中定宽
4. 交互细节：
   - 触控按钮高度 >= 44px
   - 图片懒加载与骨架屏

---

## 5. 后端架构（JDK 21 + Spring Boot 3）

### 5.1 核心依赖建议

- Web：`spring-boot-starter-web`
- 安全：`spring-boot-starter-security`
- 校验：`spring-boot-starter-validation`
- 数据库：`spring-boot-starter-data-jpa`（或 MyBatis-Plus）
- 缓存：`spring-data-redis`
- 监控：`spring-boot-starter-actuator`
- 文档：`springdoc-openapi`
- 测试：`spring-boot-starter-test`

### 5.2 模块划分建议

1. `auth`：登录、刷新令牌、权限校验
2. `user`：用户资料、状态、封禁
3. `category`：板块与导航
4. `post`：帖子内容、状态流转
5. `comment`：评论与回复树
6. `interaction`：点赞、收藏、关注
7. `notification`：站内通知
8. `moderation`：举报、审核、违规处理
9. `media`：MinIO 上传、预签名 URL
10. `search`：检索聚合（可先数据库检索，后接 ES）

### 5.3 并发与性能要点

1. 热门列表缓存到 Redis（短 TTL + 主动失效）
2. 帖子计数类字段（浏览/点赞/评论数）可异步聚合更新
3. 分页统一采用游标或 `offset + limit`（MVP 可先后者）
4. 慢查询日志与 SQL 索引治理常态化

---

## 6. 数据存储设计

### 6.1 PostgreSQL 核心表

建议包含：

- `users`
- `roles`
- `user_roles`
- `categories`
- `posts`
- `comments`
- `post_likes`
- `post_favorites`
- `notifications`
- `reports`
- `audit_logs`

关键约束建议：

1. 点赞/收藏唯一索引：`(user_id, post_id)`
2. 高频查询索引：
   - `posts(category_id, created_at desc)`
   - `posts(author_id, created_at desc)`
   - `comments(post_id, created_at asc)`
3. 软删除字段：`deleted_at`（帖子/评论）

### 6.2 Redis 使用建议

1. 验证码与登录风控计数
2. 热门帖子列表缓存
3. 用户会话或黑名单 token 缓存
4. 限流计数器（IP、用户维度）

---

## 7. 文件存储设计（自建 MinIO）

### 7.1 上传流程

推荐“后端签发预签名 URL + 客户端直传 MinIO”：

1. 前端请求上传凭证（文件类型、大小、目录）
2. 后端校验后生成 MinIO 预签名 URL
3. 前端直传 MinIO
4. 前端回调后端记录资源元数据（URL、hash、大小、业务关联）

优势：

- 减轻后端带宽压力
- 降低大文件上传超时风险
- 更适合横向扩展

### 7.2 桶与目录规划

- Bucket：`forum-media`
- 路径建议：
  - `avatars/{userId}/xxx.png`
  - `posts/{postId}/images/xxx.webp`
  - `attachments/{yyyy}/{MM}/xxx.pdf`

### 7.3 安全策略

1. Bucket 默认私有
2. 下载采用时效性预签名 URL（例如 5~30 分钟）
3. 上传白名单：MIME、后缀、大小限制
4. 文件病毒扫描（可异步接入）
5. 资源访问日志留存

---

## 8. 接口规范与鉴权

### 8.1 API 风格

- RESTful
- 统一路径前缀：`/api/v1`
- 返回结构统一：
  - `code`
  - `message`
  - `data`
  - `requestId`

### 8.2 鉴权模型

1. Access Token（短期）+ Refresh Token（长期）
2. Spring Security + JWT Filter
3. RBAC：普通用户、版主、管理员
4. 关键管理操作二次校验与审计日志

### 8.3 幂等与防刷

1. 发帖/评论可使用 `Idempotency-Key`
2. 登录、发帖、评论接口接入限流
3. 举报接口增加频次限制与风控

---

## 9. 安全与合规

1. 密码使用 `BCrypt/Argon2` 哈希存储
2. 输入参数严格校验，避免脏数据入库
3. XSS 防护（输出转义 + 富文本白名单）
4. SQL 注入防护（参数化查询）
5. CSRF 防护（Cookie 场景启用）
6. 敏感词与违规内容审核机制
7. 管理员操作全量审计

---

## 10. 可观测性与运维

### 10.1 监控

- Actuator 暴露健康检查、线程池、JVM 指标
- Prometheus + Grafana 监控：
  - QPS、P95/P99、错误率
  - JVM（堆内存、GC、线程）
  - DB 连接池、慢 SQL
  - Redis 命中率
  - MinIO 请求与错误率

### 10.2 日志

- 结构化日志（JSON）
- `requestId` 全链路透传
- 错误日志告警（如 ELK/Loki + Alert）

### 10.3 备份恢复

1. PostgreSQL：每日全量 + 增量备份
2. MinIO：跨盘/跨机冗余与生命周期策略
3. Redis：持久化策略按业务要求配置

---

## 11. 部署架构与环境规划

### 11.1 环境

- `dev`：开发联调
- `staging`：预发布验证
- `prod`：生产环境

### 11.2 容器化部署

建议先 Docker Compose，后续可迁移 Kubernetes。

核心服务：

1. `forum-web`（前端）
2. `forum-api`（Spring Boot）
3. `postgres`
4. `redis`
5. `minio`
6. `nginx`

### 11.3 Nginx 职责

1. HTTPS 终止
2. 前端静态资源缓存
3. `/api` 路由到 Spring Boot
4. 基础限流与安全头

---

## 12. CI/CD 建议

1. 提交触发：
   - 单元测试
   - 代码质量扫描（SpotBugs/Checkstyle/Sonar）
   - 镜像构建
2. 合并主干触发：
   - 部署到 staging
   - 自动化冒烟测试
3. 生产发布：
   - 蓝绿或滚动发布
   - 异常自动回滚

---

## 13. 演进路线

### 阶段 A（MVP）

- 单体应用 + PostgreSQL + Redis + MinIO
- 支持基础发帖、评论、互动、审核

### 阶段 B（增长）

- 接入 ES/Meilisearch 提升检索
- 引入消息队列处理通知与异步任务
- 热点数据与计数异步化

### 阶段 C（规模化）

- 按业务域拆分服务（内容、互动、用户）
- 灰度发布、服务网格、全链路追踪

---

## 14. 风险与规避

1. **内容审核压力**：通过“机器预审 + 人工复审”降低违规风险
2. **热点帖子读写放大**：Redis 缓存 + 异步计数聚合
3. **对象存储带宽压力**：CDN 分发 + 图片压缩与格式转换
4. **单体复杂度上升**：严格模块边界与代码规范，预留拆分接口

---

## 15. 交付物清单（建议）

1. 本架构文档（当前文件）
2. 接口文档（OpenAPI）
3. 数据库建模文档（ER 图 + DDL）
4. 部署文档（Docker Compose/K8s）
5. 运维手册（告警、备份、回滚）

