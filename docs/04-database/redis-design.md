# Redis Key 规范与连接路由设计

本文档定义 Redis 在 Next-IM 中的统一命名空间、在线设备连接路由模型、TTL 约定与防误删/跨租户隔离规范（对应任务 4、9、10、13）。

## 1. Key 命名规范

所有 Redis Key 必须遵循全局前缀与多租户命名空间约定，禁止裸写无命名空间的业务 Key：

- **租户业务 Key**：`nextim:{tenant_id}:{app_id}:{module}:{entity}:{identifier}`
- **平台/基础设施 Key**（无租户归属，如网关节点心跳）：`nextim:sys:{module}:{entity}:{identifier}`

规则：
- 分隔符统一使用冒号 `:`（`ADR-0002` 已禁止业务 ID 包含 `:`）。
- 所有动态 Key 必须设置明确的 TTL，严禁产生永不过期的僵尸连接或孤儿路由数据。
- 所有 Redis 操作封装在 `pkg/redis` 与对应模块的 `repository/` 中，强制通过 Helper 函数拼接带 `TenantContext` 的 Key。

---

## 2. 在线连接与设备路由模型（核心）

当设备通过 WebSocket Gateway 完成 `LOGIN` 鉴权后，需要在 Redis 中登记路由，供消息分发 Worker 查询目标用户当前在线的所有设备及所在 Gateway 节点。

### 2.1 用户在线设备路由表（Hash）
- **Key**：`nextim:{tenant_id}:{app_id}:conn:user:{user_id}`
- **类型**：`HASH`
- **Field**：`{device_id}`
- **Value**（JSON）：
  ```json
  {
    "conn_id": "conn_01jf8m...",
    "gateway_id": "gw-hz-01",
    "gateway_addr": "10.0.1.12:9090",
    "platform": 1,
    "connected_at_ms": 1732500000000,
    "last_heartbeat_ms": 1732500030000
  }
  ```
- **TTL**：与心跳超时绑定，单设备活跃刷新时同步延长整个 Hash 的兜底过期时间（例如 `3 * heartbeat_timeout`，默认 `180s`）。
- **过期待清理字段判断**：由于 Redis Hash 单 Field 过期依赖 Redis 7.4+ `HEXPIRE` 特性，为兼容 Redis 6.x / 7.0+，路由读取方必须检查 Value 中的 `last_heartbeat_ms` 是否在 `now - route_ttl` 内；若单个 Field 已超时，读取方忽略该设备并可异步触发惰性清理（`HDEL`）。

### 2.2 设备连接详情（String，可选精确 TTL）
- **Key**：`nextim:{tenant_id}:{app_id}:conn:dev:{user_id}:{device_id}`
- **类型**：`STRING`（存储当前生效的 `conn_id` 与 `gateway_id`）
- **TTL**：`90s`（默认为心跳间隔 `30s` 的 3 倍）。
- **用途**：
  1. 每次收到 `HEARTBEAT` 时执行 `EXPIRE` 续期。
  2. 断连注销时使用 Lua 脚本进行 **CAS 删除**：仅当 Redis 中当前记录的 `conn_id` 等于待注销连接的 `conn_id` 时才执行删除（同时 `HDEL` 用户 Hash 对应 Field），**防止旧连接延迟断开误删同设备刚重连成功的新连接路由**！

```lua
-- 安全注销连接路由 Lua 脚本 (KEYS[1]=devKey, KEYS[2]=userHashKey, ARGV[1]=deviceID, ARGV[2]=expectedConnID)
local cur = redis.call("GET", KEYS[1])
if cur == ARGV[2] then
  redis.call("DEL", KEYS[1])
  redis.call("HDEL", KEYS[2], ARGV[1])
  return 1
end
return 0
```

### 2.3 Gateway 节点存活注册表
- **Key**：`nextim:sys:gw:node:{gateway_id}`
- **类型**：`STRING`（JSON：节点内网 RPC/推送地址、启动时间、当前连接数、版本）
- **TTL**：`30s`（Gateway 每 `10s` 续约一次）。
- **用途**：Worker 分发消息前可检查目标 `gateway_id` 是否仍存活；若节点宕机且 TTL 过期，直接跳过投递并转离线同步/离线推送，避免无效网络超时。

### 2.4 Gateway 节点下属连接索引（可选，用于优雅下线）
- 单个 Gateway 进程内存中直接维护本机 `conn_id -> Connection` 映射；进程收到 `SIGTERM` 优雅关闭时，主动关闭连接并批量执行上述 CAS 注销脚本清理 Redis 路由。

---

## 3. 认证、安全与限流 Key

| 用途 | Key 格式 | 类型 | TTL | 说明 |
|---|---|---|---|---|
| Access Token 吊销黑名单 | `nextim:{tid}:{aid}:auth:revoked:{jti}` | `STRING` (`"1"`) | 剩余 Access Token 有效期 | 登出或踢下线时写入，JWT 中间件校验 |
| 分布式登录/接口限流 | `nextim:{tid}:{aid}:rl:{rule}:{subject}` | `STRING` / `ZSET` | 窗口时长（如 `60s`） | 滑动窗口或令牌桶限流 |
| App 元数据缓存 | `nextim:sys:app:meta:{app_id}` | `STRING` (JSON) | `300s` | 缓存 `tenant_id`、状态与非明文配置，降低鉴权打库压力 |
| 投递去重/重试状态（可选） | `nextim:{tid}:{aid}:dlv:inflight:{message_id}:{device_id}` | `STRING` | `60s` | 追踪等待 `MESSAGE_DELIVERED_ACK` 的在途消息 |

---

## 4. P0 初期无 Kafka 时的 Gateway 内部投递通道（可选过渡）

依据 `ADR-0001`，初期若未部署 Kafka，Outbox Worker 向目标 Gateway 投递实时消息支持两种方式（通过统一 `MessageDispatcher` 接口抽象）：
1. **单机一体化模式**：`im-gateway` 与 `im-worker` 同进程运行，直接调用内存 `ConnectionManager.PushToDevice`。
2. **多节点轻量模式（Redis Pub/Sub）**：
   - Channel：`nextim:sys:gw:push:{gateway_id}`
   - Worker 查出目标设备所在 `gateway_id` 后，向对应 Gateway 专属 Channel 发布序列化后的 `PushMessage` 载荷。
   - **重要边界**：Redis Pub/Sub 仅做在线实时通知投递（Best-effort），绝不承担持久化职责；若目标节点断连或丢包，客户端通过重连后的 `SYNC_REQUEST` 从 MySQL 游标补齐。
