# ADR-0002：统一 ID 规范、租户隔离模型与 Go 依赖基线

- 状态：Accepted
- 日期：2026-11-25
- 关联文档：`docs/04-database/data-model.md`、`docs/05-server/go-server-guide.md`、`docs/07-security/security-baseline.md`

## 1. 背景

在进入服务端与协议编码前，需要消除以下可能引发跨模块不一致的未决设计点：
1. `app_id` 是否全局唯一，以及核心表唯一索引是否包含 `tenant_id`。
2. 各实体的 ID 格式、长度与生成方式（包括单聊与群聊 `conversation_id` 的构造规则）。
3. Go 服务端首批任务（配置、日志、HTTP、WebSocket、MySQL、Migration、Redis、JWT、Protobuf）的基础库选型，避免不同任务引入互相冲突的框架。

## 2. 决策

### 2.1 租户与 `app_id` 唯一性模型
- `tenant_id` 标识企业/组织租户，全局唯一。
- `app_id` 标识租户下的独立 IM 应用，**平台全局唯一**，且归属于唯一一个 `tenant_id`（一旦创建不可跨租户转移）。
- **表字段要求**：所有业务表仍必须同时存储 `tenant_id` 与 `app_id`，以满足租户级审计、配额计费、大租户独立分库与安全防线（双字段校验）。
- **索引规则**：因 `app_id` 全局唯一，业务主查询与唯一约束优先以 `app_id` 作为最左前缀（如 `(app_id, sender_id, client_msg_id)`、`(app_id, conversation_id, seq)`），并在租户级运营查询表增加 `(tenant_id, app_id, ...)` 索引。所有仓储层查询必须由可信上下文同时注入 `tenant_id` 与 `app_id` 校验或限定。

### 2.2 统一 ID 格式规范

| 实体 | 字段名 | 类型与长度 | 生成策略 | 示例 |
|---|---|---|---|---|
| 租户 | `tenant_id` | `VARCHAR(32)` | 前缀 `tnt_` + ULID/随机 Base32 | `tnt_01jf8k3m2a9p4q7r` |
| 应用 | `app_id` | `VARCHAR(32)` | 前缀 `app_` + ULID/随机 Base32 | `app_01jf8k4n8b2c5d6e` |
| 用户 | `user_id` | `VARCHAR(64)` | 业务方外部 ID 或平台生成 `usr_` + ULID；在同一 `app_id` 内唯一 | `usr_10001` / `ext_u_42` |
| 设备 | `device_id` | `VARCHAR(64)` | 客户端持久化生成的稳定设备标识（推荐 UUIDv4/ULID），在同一 `(app_id, user_id)` 内唯一 | `dev_550e8400e29b` |
| 群组 | `group_id` | `VARCHAR(64)` | 前缀 `grp_` + ULID，在 `app_id` 内唯一 | `grp_01jf8k7x9y0z1a2b` |
| 单聊会话 | `conversation_id` | `VARCHAR(160)` | **确定性派生**：`c2c:{min(uid1,uid2)}:{max(uid1,uid2)}`（按字典序排序；`user_id` 禁止包含 `:`） | `c2c:usr_10001:usr_10002` |
| 群聊会话 | `conversation_id` | `VARCHAR(160)` | **确定性派生**：`grp:{group_id}` | `grp:grp_01jf8k7x9y0z1a2b` |
| 服务端消息 | `message_id` | `BIGINT UNSIGNED`（对外序列化为字符串或 `uint64`） | 64 位 Snowflake（41 位毫秒时间戳 + 10 位节点 ID + 12 位序列号），趋势递增、全局唯一 | `728391048291041281` |
| 客户端幂等 ID | `client_msg_id` | `VARCHAR(64)` | 客户端发送前生成的 UUIDv4 / ULID，重试必须复用 | `01jf8m0099aa11bb` |
| 会话序号 | `seq` | `BIGINT UNSIGNED` | 会话级单调严格递增整数，从 `1` 开始，由数据库原子递增分配 | `1, 2, 3, ...` |
| Outbox 事件 | `event_id` | `VARCHAR(36)` | UUIDv7 / ULID，全局唯一 | `01jf8m01aabbccdd` |
| 请求追踪 | `request_id` | `VARCHAR(64)` | `req_` + ULID（或透传上游合规 `X-Request-ID`） | `req_01jf8m02eeff0011` |

约束补充：
- 禁止在 `user_id`、`tenant_id`、`app_id`、`group_id` 中包含冒号 `:` 或空白字符，防止与 `conversation_id` 和 Redis Key 分隔符冲突；入口校验需限制字符集为 `[a-zA-Z0-9_-]`，长度 1~64。
- JSON / REST 响应中，64 位整型 `message_id` 和 `seq` 若存在 JavaScript 精度风险：`message_id` 统一以 **字符串** 表示（避免 JS `Number.MAX_SAFE_INTEGER` 溢出），`seq` 在安全整数范围内按 `int64` / `uint64` 输出（Protobuf 使用 `uint64` / `int64`）。

### 2.3 Go 服务端技术栈与依赖基线（P0）

为控制复杂度并保持标准库优先原则，Go 服务端（`next-im-server`）确定以下基础选型：

| 能力 | 选型 | 说明 |
|---|---|---|
| Go 版本 | Go 1.23+ | 使用标准库 `log/slog`、`net/http` 增强路由等特性 |
| 结构化日志 | 标准库 `log/slog`（JSON Handler） | 零外部依赖，封装脱敏与 `ctx` 字段自动注入 |
| HTTP 路由 | `github.com/go-chi/chi/v5`（或 Go 1.22+ 标准 `net/http.ServeMux`） | 优先 `chi/v5`（标准 `http.Handler` 兼容、轻量、中间件生态成熟） |
| WebSocket | `github.com/coder/websocket`（或 `github.com/gorilla/websocket`） | 优先 `github.com/coder/websocket`（原生支持 `context.Context` 与并发写安全封装） |
| 配置加载 | `gopkg.in/yaml.v3` + 环境变量覆盖（`NEXTIM_` 前缀） | 轻量可控，不引入重型全局单例配置框架 |
| MySQL 访问 | `database/sql` + `github.com/go-sql-driver/mysql` + `sqlx`（可选） | 显式 SQL 与清晰事务边界，避免黑盒 ORM 隐藏多租户条件 |
| Migration | `github.com/golang-migrate/migrate/v4` 或 `github.com/pressly/goose/v3` | 优先 `goose/v3`（支持 SQL 文件嵌入 `embed.FS` 与 Go 代码迁移） |
| Redis 客户端 | `github.com/redis/go-redis/v9` | 官方主流客户端，原生 `context.Context` 支持 |
| JWT | `github.com/golang-jwt/jwt/v5` | 支持 HS256/RS256/EdDSA，严格校验算法与声明 |
| Protobuf 工具链 | `buf` + `google.golang.org/protobuf` | 统一 lint、breaking change 检查与多语言代码生成 |
| 单元测试 | 标准库 `testing` + `github.com/stretchr/testify` | 清晰断言与 table-driven tests |

## 3. 后果

- 正面：消除任务 2~16 中的所有关键歧义；单聊会话 ID 确定性派生免去额外的“查两人是否已有会话”竞态查询；`database/sql` 显式 SQL 便于严格审查多租户条件与原子 `seq` 事务。
- 约束：一旦发布，ID 字符集规则、单聊 `conversation_id` 拼接规则与 Protobuf 包路径不可随意变更。若未来调整选型，需新增取代本记录的 ADR。
