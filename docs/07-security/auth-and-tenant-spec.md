# 认证、授权与多租户上下文规范

本文档细化 `security-baseline.md`，为任务 5（tenant/app 上下文与 HTTP 中间件）、任务 8（App 认证、用户登录、Token 签发和刷新）和任务 9（WS Gateway 鉴权）提供可直接落地的实现约定。

## 1. 可信租户与应用上下文 (`internal/tenant`)

所有业务 Service 和 Repository 只能从 `context.Context` 读取由鉴权中间件构建的不可变结构体：

```go
type AuthContext struct {
    TenantID  string // 可信租户 ID（绝不取自客户端自由头）
    AppID     string // 可信应用 ID
    UserID    string // 终端用户 ID（Server-to-Server 开放接口可为空）
    DeviceID  string // 终端设备 ID（Server-to-Server 接口可为空）
    SessionID string // 当前认证会话 ID (sid/jti)
    Principal string // "user" | "app_server" | "admin"
    RequestID string // 链路追踪 ID
}
```

强制约束：
- 提供 `WithAuthContext(ctx, authCtx)` 和 `MustFromContext(ctx)` / `FromContext(ctx) (AuthContext, bool)`。
- 业务 Repository 方法接收 `ctx`，内部必须提取 `AuthContext`；若缺失可信上下文直接返回 `10002 UNAUTHORIZED` 或编程契约错误，防止开发疏漏造成全表越权查询。
- 单元测试与集成测试必须包含 **跨租户 / 跨 App 越权访问用例**：构造租户 A 的 `AuthContext` 去读/写租户 B 的 `user_id`、`conversation_id`、`message_id`，验证返回 `403/404` 且数据零泄露。

---

## 2. HTTP 中间件执行链路（顺序固定）

1. **RequestID 中间件**：校验或生成 `req_<ulid>`，注入 `ctx` 并写入响应头 `X-Request-ID`。
2. **Recovery 中间件**：捕获 panic，记录堆栈日志，返回安全 `50001 INTERNAL_ERROR`。
3. **Structured Access Log 中间件**：记录方法、脱敏路径、状态码、耗时、业务错误码、`request_id`。
4. **限流中间件（RateLimit）**：按 IP / AppID / UserID 保护入口。
5. **鉴权与上下文注入中间件**：
   - **终端用户路由（`/api/v1/...`）**：解析 `Authorization: Bearer <access_token>` -> 验签、验过期、验 `iss`、查 Redis 吊销黑名单 -> 校验 App / Tenant 状态为 active -> 构造 `AuthContext` 写入 `ctx`。若请求同时携带 `X-App-ID` 且与 JWT 中 `aid` 不符，立即拒绝（`10003 FORBIDDEN`）。
   - **服务端开放路由（`/open/v1/...` 或服务端代登入口）**：验证 App 签名 -> 查 `im_app` 验证应用与租户状态 -> 构造 `Principal = "app_server"` 的 `AuthContext`。

---

## 3. JWT Access Token 与 Refresh Token 机制

### 3.1 Access Token (JWT)
- 算法：P0 默认 `HS256`（后续支持 `RS256` / `EdDSA` 及 `kid` 密钥轮换），必须白名单校验 `alg`，防止 `alg: none` 攻击。
- 有效期：默认 `2 小时`（可按应用配置在 15m ~ 24h 之间调整）。
- Standard + Custom Claims：
  ```json
  {
    "iss": "next-im",
    "sub": "usr_10001",
    "jti": "tok_01jf8m...",
    "sid": "ses_01jf8m...",
    "tid": "tnt_01jf8k...",
    "aid": "app_01jf8k...",
    "did": "dev_550e8400e29b",
    "plt": 1,
    "iat": 1732500000,
    "exp": 1732507200
  }
  ```

### 3.2 Refresh Token 与轮换（Rotation）
- 格式：高熵随机不透明字符串（至少 256-bit 随机数 Base64URL 编码，前缀 `rt_`），数据库 `im_auth_session` 仅存 `SHA-256(refresh_token)`。
- 有效期：默认 `30 天`。
- **轮换与重放检测**：
  - 每次调用 `/api/v1/auth/refresh` 时，必须在数据库事务中核对 `refresh_token_hash`、`device_id`、`revoked_at IS NULL` 与 `expires_at > NOW()`。
  - 刷新成功后，签发新的 `access_token` 与新的 `refresh_token`，原地原子更新 `refresh_token_hash`（或作废旧记录插入新记录）。
  - 若已作废的旧 `refresh_token` 再次被提交（疑似令牌泄漏重放），直接吊销该 `(app_id, user_id, device_id)` 的全部活跃会话并返回 `10008 REFRESH_TOKEN_INVALID`。

---

## 4. 服务端开放 API 签名认证（AppID + AppSecret）

第三方业务后端调用 Next-IM 开放接口（如服务端注册用户、签发登录凭证、发送系统消息）时，严禁直接明文传输 `AppSecret`，统一采用 HMAC-SHA256 请求签名：

- 请求头：
  - `X-App-ID`：`app_xxx`
  - `X-Timestamp`：UTC 秒级时间戳（允许与服务端时钟偏差 `±300s`，超时拒绝防重放）
  - `X-Nonce`：每次请求唯一的随机字符串（至少 16 字符，服务端用 Redis 短期记录防 5 分钟内重放）
  - `X-Signature`：`hex(HMAC_SHA256(app_secret, canonical_string))`
- 签名原文 `canonical_string`（换行符 `\n` 拼接）：
  ```text
  {HTTP_METHOD}
  {REQUEST_PATH}
  {X-App-ID}
  {X-Timestamp}
  {X-Nonce}
  {SHA256_HEX(REQUEST_BODY)}
  ```
- 校验使用常量时间比较（`hmac.Equal`），防止时序侧信道攻击。

---

## 5. 多设备在线与互踢策略

应用可通过 `im_app.config_json` 配置多端登录策略（P0 默认启用 **同端互斥、跨端共存**）：
- `multi_device_same_platform`：允许同平台多设备同时在线（如两台 iOS 同时在线）。
- `single_device_per_platform`（默认）：同平台（如 iOS）新设备登录时，保留 Android / Web / Desktop 在线，但向旧 iOS 设备下发 `COMMAND_KICK_DEVICE`（`KICK_REASON_SAME_PLATFORM_LOGIN`），吊销旧设备 Session 并断开旧连接。
- `single_device_global`：单用户全局只允许一台设备在线。
