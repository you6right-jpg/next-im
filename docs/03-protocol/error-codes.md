# 错误码注册表

以下为权威错误码注册表，所有模块必须统一引用 `pkg/errors` 中的常量，发布后保持数值和语义稳定，严禁私自重复分配。

| Code | 名称 | HTTP Status | 可重试 | 说明 |
|---:|---|---:|---|---|
| 0 | `SUCCESS` | 200 | 否 | 成功 |
| 10001 | `INVALID_ARGUMENT` | 400 | 否 | 参数无效或格式错误 |
| 10002 | `UNAUTHORIZED` | 401 | 否 | 未认证或缺少凭证 |
| 10003 | `FORBIDDEN` | 403 | 否 | 无权访问目标资源（含跨租户拦截） |
| 10004 | `NOT_FOUND` | 404 | 否 | 资源不存在 |
| 10005 | `RATE_LIMITED` | 429 | **是** | 请求触发限流，按 `retry_after_ms` 退避重试 |
| 10006 | `TOKEN_EXPIRED` | 401 | 否 | Access Token 已过期，需走刷新流程 |
| 10007 | `TOKEN_INVALID` | 401 | 否 | Access Token 签名无效或已被吊销 |
| 10008 | `REFRESH_TOKEN_INVALID` | 401 | 否 | Refresh Token 无效、已过期或已轮换失效，需重新登录 |
| 10009 | `PROTOCOL_UNSUPPORTED` | 400 | 否 | 不支持的协议版本或未知命令字 |
| 10010 | `PAYLOAD_TOO_LARGE` | 413 | 否 | 请求体或 WS 帧超过大小限制 |
| 20001 | `USER_NOT_FOUND` | 404 | 否 | 用户不存在 |
| 20002 | `USER_DISABLED` | 403 | 否 | 用户已禁用 |
| 20003 | `DEVICE_REVOKED` | 401 | 否 | 设备会话已被撤销或踢下线 |
| 30001 | `GROUP_NOT_FOUND` | 404 | 否 | 群不存在 |
| 30002 | `NOT_GROUP_MEMBER` | 403 | 否 | 不是群成员 |
| 40001 | `MESSAGE_DUPLICATE` | 200 | 否 | 重复消息（幂等命中，返回既有 `message_id` 与 `seq`） |
| 40002 | `MESSAGE_RECALLED` | 409 | 否 | 消息已撤回 |
| 40003 | `SYNC_CURSOR_INVALID` | 409 | 否 | 同步游标无效或已过期，需重建基线 |
| 40004 | `CONVERSATION_NOT_FOUND` | 404 | 否 | 会话不存在 |
| 40005 | `CONVERSATION_FORBIDDEN` | 403 | 否 | 无权向该会话发送或拉取消息 |
| 50001 | `INTERNAL_ERROR` | 500 | **是** | 内部未知错误（可按上限有限重试） |
| 50002 | `DATABASE_ERROR` | 503 | **是** | 数据库暂时不可用或执行异常 |
| 50003 | `CACHE_ERROR` | 503 | **是** | Redis 缓存或路由层异常 |
| 50004 | `SERVICE_UNAVAILABLE` | 503 | **是** | 服务过载、维护中或优雅关闭中 |
| 60001 | `FILE_NOT_FOUND` | 404 | 否 | 文件不存在 |
| 60002 | `FILE_ACCESS_DENIED` | 403 | 否 | 文件无权访问 |
| 70001 | `APP_DISABLED` | 403 | 否 | 应用已停用或不存在 |
| 70002 | `TENANT_DISABLED` | 403 | 否 | 租户已停用 |
| 70003 | `LICENSE_INVALID` | 403 | 否 | License 无效或已过期 |
| 70004 | `APP_SIGNATURE_INVALID` | 401 | 否 | 服务端开放 API 签名校验失败或时间戳过期 |

规则：
- 不通过 HTTP status 取代业务错误码；客户端业务判断必须依据响应体中的 `code`。
- 日志记录内部 `cause` 与堆栈，客户端响应体只返回安全的 `message`，禁止透传 SQL/Redis/内网地址。
- 响应体与 WS `ErrorResponse` 必须通过 `retryable` 字段显式告知客户端是否可重试；不可重试错误（如参数错误、无权限）禁止客户端自动重试。
