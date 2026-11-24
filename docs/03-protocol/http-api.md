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

```json
{
  "code": 0,
  "message": "success",
  "request_id": "req_xxx",
  "data": {}
}
```

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
