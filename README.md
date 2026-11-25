# Next-IM

商业级即时通讯平台（IMaaS / 私有化 IM）项目文档入口。

## 项目目标

建设一套可商业化交付的 IM 基础设施，提供 Go 服务端、iOS / Android / Flutter / Web SDK、管理后台、开放平台，以及 SaaS 和私有化部署能力。

## 技术决策基线

| 领域 | 决策 |
|---|---|
| 服务端 | Go，模块化单体起步，支持后续服务拆分 |
| 实时通信 | WebSocket |
| 普通 API | REST / JSON |
| 协议定义 | Protocol Buffers |
| 主数据库 | MySQL 8 |
| 缓存与连接路由 | Redis |
| 异步事件 | Transactional Outbox + Kafka |
| 文件存储 | S3 兼容存储 / MinIO |
| 搜索 | OpenSearch（按阶段引入） |
| iOS | Swift |
| Android | Kotlin |
| Flutter | Dart Plugin，首期桥接原生 SDK |
| Web | TypeScript |
| 管理后台 | Vue 3 + TypeScript |
| 部署 | Docker；规模化后 Kubernetes |
| 可观测性 | Prometheus、Grafana、OpenTelemetry、结构化日志 |

## 文档导航

- `docs/01-product/`：产品定位（`product-overview.md`）、功能优先级矩阵（`feature-matrix.md`）
- `docs/02-architecture/`：总体架构（`system-architecture.md`）、模块边界（`module-boundaries.md`）、部署拓扑（`deployment.md`）
- `docs/03-protocol/`：协议总则（`protocol-overview.md`）、WebSocket 交互规范（`websocket-protocol.md`）、Protobuf v1 详细定义（`proto-spec-v1.md`）、HTTP API 契约（`http-api.md`）、错误码注册表（`error-codes.md`）
- `docs/04-database/`：数据模型（`data-model.md`）、P0 核心 DDL 表结构（`p0-schema-spec.md`）、Redis Key 与路由设计（`redis-design.md`）、分库分表与索引策略（`sharding-strategy.md`）
- `docs/05-server/`：Go 服务端规范（`go-server-guide.md`）、配置日志错误与启停规范（`config-and-bootstrap.md`）、模块开发模板（`module-template.md`）
- `docs/06-sdk/`：跨端 SDK 架构（`sdk-architecture.md`）、iOS SDK 详细规范（`ios-sdk-spec.md`）
- `docs/07-security/`：安全基线（`security-baseline.md`）、认证授权与多租户上下文规范（`auth-and-tenant-spec.md`）
- `docs/08-reliability/`：消息可靠性（`message-reliability.md`）、同步与状态模型（`sync-and-state.md`）、Outbox 与消息分发设计（`outbox-and-delivery.md`）
- `docs/09-operations/`：部署升级与恢复（`deployment-and-upgrade.md`）、可观测性与服务目标（`observability-and-slo.md`）
- `docs/10-testing/`：测试策略与验收标准（`test-strategy.md`）
- `ai/`：AI 编码规则（`AGENTS.md`）、首批任务顺序（`first-tasks.md`）、任务模板（`task-template.md`）、代码审查清单（`code-review-checklist.md`）
- `decisions/`：架构决策记录（`ADR-0001-architecture-baseline.md`、`ADR-0002-id-tenant-and-tech-baseline.md`）

## 关键原则

1. 消息必须先持久化，再向发送端确认服务端已接受。
2. 客户端重试必须复用同一个 `client_msg_id`，服务端以唯一约束实现幂等。
3. 会话内顺序由服务端 `seq` 定义，不依赖客户端时间戳。
4. 实时推送不是消息存储；离线恢复必须通过游标同步。
5. 推送通知不是消息通道，也不能作为 Delivered 的依据。
6. 每个业务入口都必须执行租户 / 应用上下文校验。
7. 协议、数据库迁移和 SDK 公共 API 都必须版本化。
8. 初期不盲目微服务化；模块边界清晰比服务数量多更重要。
9. AI 每次只实现一个有边界、可测试、可验收的任务。
10. 性能数字必须通过明确环境下的压测验证，不写未经验证的 SLA。

## 当前状态

本仓库文档定义的是目标架构基线。具体实现时，新增或修改关键决策必须更新对应文档及 ADR。
