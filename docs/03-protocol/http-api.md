# HTTP API 规范

统一前缀：`/api/v1`

## 约定

- JSON 使用 lower_snake_case。
- 时间使用 RFC 3339 UTC，数据库按统一约定存储 UTC。
- 分页使用 cursor 或明确的 page / page_size。
- 所有请求注入 `request_id`、`tenant_id`、`app_id` 和身份上下文。
- 幂等写接口根据业务需要支持 `Idempotency-Key`。
- 错误码稳定、机器可读；message 仅用于诊断展示。

## 示例路由

```text
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
GET    /api/v1/users/me
GET    /api/v1/users/{user_id}
POST   /api/v1/friends/requests
GET    /api/v1/friends
DELETE /api/v1/friends/{user_id}
POST   /api/v1/groups
GET    /api/v1/groups/{group_id}
POST   /api/v1/groups/{group_id}/members
GET    /api/v1/conversations
GET    /api/v1/conversations/{conversation_id}/messages
POST   /api/v1/files/upload-token
GET    /api/v1/files/{file_id}
```

## 统一响应

成功响应：
```json
{
  "code": 0,
  "message": "success",
  "request_id": "req_01jf8m02eeff0011",
  "data": {}
}
```

错误响应：
```json
{
  "code": 10005,
  "message": "too many requests",
  "request_id": "req_01jf8m02eeff0011",
  "retryable": true,
  "retry_after_ms": 1000
}
```

## 标准请求头

| Header | 必填范围 | 说明 |
|---|---|---|
| `Authorization` | 用户鉴权接口 | `Bearer <access_token>`，服务端据此解析可信 `tenant_id / app_id / user_id / device_id` |
| `X-App-ID` | 登录与开放接口 | 客户端声明的应用 ID；若同时携带 JWT，必须与 JWT 中的 `aid` 一致否则返回 `403` |
| `X-Request-ID` | 可选 | 客户端链路追踪 ID；缺失时由 API Gateway 自动生成 `req_<ulid>` 并回写响应头 |
| `X-Device-ID` | 可选/登录必填 | 设备唯一标识 |
| `Idempotency-Key` | 幂等写接口 | 客户端防重提交 Key |
| `X-Timestamp` / `X-Nonce` / `X-Signature` | 服务端开放 API | 见 `docs/07-security/auth-and-tenant-spec.md` |

注意：**绝不允许信任客户端传入的 `X-Tenant-ID`**，可信 `tenant_id` 只能由服务端通过 `app_id` 查询或签名后的 JWT 推导。

## 健康检查端点（免鉴权）

- `GET /healthz`：存活探针（Liveness），仅检查进程存活，返回 `200 {"status":"ok"}`。
- `GET /readyz`：就绪探针（Readiness），校验 MySQL 与 Redis 连通性及服务未进入优雅关闭状态；依赖异常时返回 `503` 及各组件检查状态。

## P0 核心接口详细契约

### 1. 服务端签发用户登录（Server-to-Server 或开发态登录）`POST /api/v1/auth/login`
- 请求（服务端集成方代用户换取 Token，或使用业务方校验后的凭证）：
```json
{
  "user_id": "usr_10001",
  "device_id": "dev_550e8400e29b",
  "platform": 1,
  "device_name": "iPhone 16 Pro",
  "app_version": "1.0.0",
  "sdk_version": "0.1.0"
}
```
- 响应 `data`：
```json
{
  "access_token": "eyJhbGciOi...",
  "access_token_expires_in": 7200,
  "refresh_token": "rt_01jf8m...",
  "refresh_token_expires_in": 2592000,
  "token_type": "Bearer",
  "ws_url": "wss://ws.example.com/ws"
}
```

### 2. 刷新令牌 `POST /api/v1/auth/refresh`
- 请求：
```json
{
  "refresh_token": "rt_01jf8m...",
  "device_id": "dev_550e8400e29b"
}
```
- 响应 `data`：返回新的 `access_token` 与轮换后的新 `refresh_token`（旧 `refresh_token` 立即作废）。

### 3. 登出 `POST /api/v1/auth/logout`
- 鉴权：需要 `Authorization: Bearer <access_token>`。
- 行为：吊销当前 `session_id`、将当前 Access Token `jti` 写入 Redis 黑名单、断开并清理该设备的 WebSocket 连接路由。

### 4. 获取会话列表 `GET /api/v1/conversations?cursor=&limit=50`
- 响应 `data`：
```json
{
  "items": [
    {
      "conversation_id": "c2c:usr_10001:usr_10002",
      "conversation_type": 1,
      "peer_id": "usr_10002",
      "last_seq": 42,
      "read_seq": 40,
      "unread_count": 2,
      "is_pinned": false,
      "is_muted": false,
      "last_message": {
        "message_id": "728391048291041281",
        "seq": 42,
        "sender_id": "usr_10002",
        "message_type": 1,
        "content": "{\"text\":\"hello\"}",
        "created_at": "2026-11-25T08:00:00.123Z"
      }
    }
  ],
  "next_cursor": "",
  "has_more": false
}
```

### 5. 拉取会话历史消息 `GET /api/v1/conversations/{conversation_id}/messages?before_seq=42&limit=50`
- 参数：`before_seq`（向上翻页，不含该 `seq`；不传则从最新开始）、`after_seq`（向后增量拉取）、`limit`（默认 50，最大 200）。
- 权限：校验当前用户是否属于该会话，越权返回 `40005 CONVERSATION_FORBIDDEN`。

### 6. HTTP 增量同步兜底 `POST /api/v1/sync`
- 用途：在 WebSocket 尚未就绪或弱网恢复大批量补数据时提供与 WS `SYNC_REQUEST` 等价的 HTTP 增量同步接口，请求与响应结构严格对齐 `docs/03-protocol/proto-spec-v1.md` 中的 `SyncRequest` / `SyncResponse`。

## 错误码分段建议

- 0：成功
- 1xxxx：通用参数、鉴权、限流错误
- 2xxxx：用户和设备
- 3xxxx：好友和群组
- 4xxxx：消息、会话和同步
- 5xxxx：基础设施和内部错误
- 6xxxx：文件和推送
- 7xxxx：租户、应用、计费和授权

错误码必须在 `error-codes.md` 集中登记，禁止模块自行重复分配。
