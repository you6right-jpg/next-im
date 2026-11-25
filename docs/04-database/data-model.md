# 数据模型设计

## 1. 全局约束

- 业务数据必须能追溯到 `tenant_id` 和 `app_id`。平台级表可按实际归属设计。
- 所有查询通过租户上下文执行，不允许漏带隔离条件。
- 使用统一 ID 生成策略；避免各模块各自生成不同格式 ID。
- 时间统一 UTC。
- 所有 schema 变更使用 Migration。
- 消息附件存对象存储，数据库只保存元数据和 storage_key。
- 软删除字段和物理删除策略必须逐表定义。
- 热点表、保留周期和归档策略需在容量评估后确定。

## 2. 核心表清单

| 表 | 作用 |
|---|---|
| im_tenant | 租户 |
| im_app | 应用、AppID、密钥摘要、配置 |
| im_user | 用户 |
| im_device | 设备 |
| im_connection / Redis route | 在线连接状态；连接瞬态优先放 Redis |
| im_friend | 好友关系 |
| im_friend_request | 好友申请 |
| im_group | 群组 |
| im_group_member | 群成员 |
| im_conversation | 会话定义 |
| im_conversation_user | 用户维度会话状态、read_seq |
| im_message | 消息主体 |
| im_message_user_state | 用户维度送达 / 已读状态（按产品策略） |
| im_user_sync | 用户设备同步游标 |
| im_outbox | 事务发件箱 |
| im_push_task | 推送任务 |
| im_file | 文件元数据 |
| im_webhook | Webhook 配置 |
| im_webhook_delivery | Webhook 投递记录 |
| im_audit_log | 管理审计 |
| im_usage_record | 用量事件或聚合记录 |
| im_license | 私有化授权 |

## 3. 消息字段建议

- `message_id`：服务端消息 ID。
- `client_msg_id`：发送端生成的幂等 ID。
- `app_id` / `tenant_id`：隔离上下文。
- `conversation_id`：所属会话。
- `seq`：会话内单调递增序号。
- `sender_id`：发送者。
- `message_type`：消息类型。
- `content`：消息载荷或结构化内容。
- `created_at`：服务端创建时间。
- `recalled_at`：撤回时间（如适用）。

建议约束：
- `(app_id, sender_id, client_msg_id)` 唯一。
- `(app_id, conversation_id, seq)` 唯一。
- `(app_id, message_id)` 唯一。

已定稿约定（见 `decisions/ADR-0002-id-tenant-and-tech-baseline.md`）：
- `app_id` 为平台全局唯一且归属于唯一 `tenant_id`，因此上述唯一索引以 `app_id` 为最左前缀即可保证隔离唯一性。
- 所有业务表仍必须同时存储 `tenant_id` 与 `app_id`，仓储层查询必须注入可信上下文的双字段条件。

## 4. 序列分配

会话 `seq` 必须单调递增。P0 阶段在 `im_conversation` 表维护 `last_seq`，在消息持久化同一事务中通过行锁（`SELECT ... FOR UPDATE` + `UPDATE`）原子递增分配；严禁使用 `MAX(seq)+1`。具体表 DDL 与事务 SQL 见 `docs/04-database/p0-schema-spec.md`。

## 5. 容量与扩展

- 初期 MySQL 单库或主从即可。
- 消息表增长后，依据租户、会话和时间分布评估分区 / 分表，详见 `docs/04-database/sharding-strategy.md`。
- 在线连接路由与缓存 Key 命名见 `docs/04-database/redis-design.md`。
- 分表路由规则一旦发布需长期兼容。
- OpenSearch 是索引，不是消息事实数据源。
- 归档、删除和合规保留周期必须有明确策略。
