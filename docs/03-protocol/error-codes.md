# 错误码注册表

以下为初始分段建议，具体 code 应在实现前集中登记，发布后保持稳定。

| Code | 名称 | 说明 |
|---:|---|---|
| 0 | SUCCESS | 成功 |
| 10001 | INVALID_ARGUMENT | 参数无效 |
| 10002 | UNAUTHORIZED | 未认证 |
| 10003 | FORBIDDEN | 无权限 |
| 10004 | NOT_FOUND | 资源不存在 |
| 10005 | RATE_LIMITED | 请求限流 |
| 20001 | USER_NOT_FOUND | 用户不存在 |
| 20002 | USER_DISABLED | 用户已禁用 |
| 30001 | GROUP_NOT_FOUND | 群不存在 |
| 30002 | NOT_GROUP_MEMBER | 不是群成员 |
| 40001 | MESSAGE_DUPLICATE | 重复消息，返回既有结果 |
| 40002 | MESSAGE_RECALLED | 消息已撤回 |
| 40003 | SYNC_CURSOR_INVALID | 同步游标无效或已过期 |
| 50001 | INTERNAL_ERROR | 内部错误 |
| 50002 | DATABASE_ERROR | 数据库错误 |
| 60001 | FILE_NOT_FOUND | 文件不存在 |
| 60002 | FILE_ACCESS_DENIED | 文件无权访问 |
| 70001 | APP_DISABLED | 应用已停用 |
| 70002 | TENANT_DISABLED | 租户已停用 |
| 70003 | LICENSE_INVALID | License 无效 |

规则：
- 不通过 HTTP status 取代业务错误码。
- 日志记录内部 cause，客户端只收到安全的错误信息。
- 重试类错误必须明确标识是否可重试。
