# Go 服务端配置、日志、错误与启动生命周期规范

本文档直接服务于 `ai/first-tasks.md` 任务 2（工程骨架、配置、日志、统一错误）和任务 3（MySQL、Migration、健康检查），确保所有二进制入口（`im-api`、`im-gateway`、`im-worker`、`im-migrate`）遵循统一基线。

## 1. 配置模型与加载规则

### 1.1 加载优先级（从低到高）
1. 代码内安全默认值（仅限非敏感连接超时、端口、日志级别等）。
2. YAML 配置文件（通过 `-config` 参数或 `NEXTIM_CONFIG` 环境变量指定，例如 `configs/local.yaml`）。
3. 环境变量覆盖（统一前缀 `NEXTIM_`，使用双下划线或明确键名映射，如 `NEXTIM_MYSQL_DSN`、`NEXTIM_REDIS_ADDR`、`NEXTIM_JWT_SECRET`）。
4. **启动期强校验**：缺少必填密钥或非法参数时，进程必须在监听端口前直接 Fail-fast 并输出清晰错误，禁止带着空密钥启动。

### 1.2 标准配置结构示例 (`configs/config.example.yaml`)

```yaml
app:
  env: local                 # local | test | staging | production | private
  node_id: 1                 # Snowflake 节点 ID (0~1023)
  instance_id: gw-local-01   # 实例唯一标识（用于 Gateway 路由与 Outbox 租约）

log:
  level: info                # debug | info | warn | error
  format: json               # json | text（仅 local 可用 text）

http:
  addr: ":8080"
  read_timeout: 10s
  write_timeout: 15s
  idle_timeout: 60s
  shutdown_timeout: 15s

ws:
  addr: ":8081"
  path: "/ws"
  login_timeout: 5s
  heartbeat_interval: 30s
  heartbeat_timeout: 90s
  max_frame_bytes: 65536     # 64 KB
  write_queue_size: 256      # 单连接异步发送缓冲区长度

mysql:
  dsn: ""                    # 必须通过环境变量或本地未提交配置注入
  max_open_conns: 50
  max_idle_conns: 10
  conn_max_lifetime: 30m
  conn_max_idle_time: 5m

redis:
  addr: "127.0.0.1:6379"
  password: ""               # 通过环境变量注入
  db: 0
  pool_size: 50
  dial_timeout: 3s
  read_timeout: 2s
  write_timeout: 2s

auth:
  jwt_issuer: "next-im"
  jwt_secret: ""             # 至少 32 字节，严禁写入 Git
  access_token_ttl: 2h
  refresh_token_ttl: 720h    # 30 天

outbox:
  poll_interval: 500ms
  batch_size: 100
  lease_duration: 30s
  max_retries: 10
```

---

## 2. 结构化日志与脱敏规范 (`pkg/logger`)

- 基于标准库 `log/slog` 封装，生产环境强制输出 JSON 格式，时间统一 UTC RFC3339Nano。
- 提供 `FromContext(ctx)` 自动附带请求公共字段：`request_id`、`trace_id`、`tenant_id`、`app_id`、`user_id`、`device_id`、`conn_id`。
- **自动脱敏拦截器（ReplaceAttr）**：以下字段名（忽略大小写或包含子串）严禁打印明文，统一替换为 `"[REDACTED]"`：
  - `password`、`secret`、`app_secret`、`token`、`access_token`、`refresh_token`、`authorization`、`dsn`、`cookie`、`content`（消息正文默认不打 Info 日志）。

---

## 3. 统一错误模型 (`pkg/errors`)

所有业务与基础设施错误统一使用 `AppError` 结构体表达，确保 HTTP API 与 WebSocket ErrorResponse 语义一致：

```go
type AppError struct {
    Code         int    // 对齐 docs/03-protocol/error-codes.md
    HTTPStatus   int    // 建议的 HTTP 状态码
    Message      string // 暴露给客户端的安全提示信息
    Retryable    bool   // 是否允许客户端自动退避重试
    RetryAfterMs uint32 // 可选，限流/过载场景下的等待建议
    Cause        error  // 内部真实错误链（仅记录日志，绝不序列化返回给客户端）
}
```

要求：
- 实现 `Error() string` 与 `Unwrap() error`，兼容标准库 `errors.Is` / `errors.As`。
- 提供工厂方法：`New(code, msg)`、`Wrap(err, code, msg)`、`FromError(err)`（将未知 error 兜底映射为 `50001 INTERNAL_ERROR` 并隐藏原始错误信息）。
- 单元测试需验证：错误链透传、`Cause` 不会序列化到 JSON 响应、所有错误码与 `error-codes.md` 一致。

---

## 4. 优雅启动与关闭生命周期

所有服务进程（`im-api`、`im-gateway`、`im-worker`）必须统一监听 `SIGINT`、`SIGTERM` 信号并按以下顺序优雅退出：

1. **标记非就绪**：将 `/readyz` 状态切换为 `draining`（返回 `503`），通知负载均衡停止转发新流量。
2. **停止接收新任务/新请求**：
   - `im-api`：调用 `http.Server.Shutdown(ctx)` 等待在途请求完成。
   - `im-gateway`：停止接受新 WebSocket 升级，向在连客户端发送优雅关闭帧并执行 Redis 路由 CAS 注销。
   - `im-worker`：取消根 `context.Context`，停止领取新 Outbox 批次，等待当前在途事件处理完成。
3. **释放基础设施连接**：按逆序关闭 Redis 客户端、MySQL 连接池并 Flush 日志与指标。
4. **超时保护**：总关闭时间受 `shutdown_timeout` 约束，超时后记录未完成任务告警并强制退出。
