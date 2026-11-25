# iOS SDK 详细设计规范（P0）

本文档细化 `sdk-architecture.md`，指导 `next-im-sdk-ios` 仓库的工程组织、连接状态机、本地 SQLite 存储结构与公共 API 契约（对应任务 17）。

## 1. 工程与包结构（Swift Package）

- 最低系统版本：iOS 15.0+，Swift 5.9+（全面采用 Swift Concurrency `async/await` 与 `actor` 保证线程安全）。
- 依赖极简原则：优先使用苹果原生框架（`Foundation` / `URLSessionWebSocketTask` / `Security` Keychain / `Network` `NWPathMonitor`），SQLite 封装可选 `GRDB.swift` 或轻量 `SQLite3` C 桥接，Protobuf 使用 `swift-protobuf`。

```text
next-im-sdk-ios/
├── Package.swift
├── Sources/
│   └── NextIMSDK/
│       ├── Public/            # 稳定公共 API、模型、错误类型、Delegate/AsyncStream
│       ├── Core/              # NextIMClient、状态协调器、依赖组装
│       ├── Auth/              # Token 管理、Refresh 串行化、Keychain 存储
│       ├── Transport/         # HTTPClient、WebSocketTransport、Envelope 编解码
│       ├── Connection/        # 连接状态机、心跳、退避重连、网络监听
│       ├── Storage/           # SQLite Manager、Migration Runner、DAO
│       ├── Message/           # 发送队列、幂等重试、去重落库、ACK 管理
│       ├── Conversation/      # 会话列表与未读数派生管理
│       └── Sync/              # 连续游标推进、缺口检测、重连增量同步
└── Tests/
    └── NextIMSDKTests/
```

---

## 2. 连接状态机

UI 层只订阅 SDK 发布的 `ConnectionState`，严禁自行推断连接状态：

```text
disconnected
   │  login() / connect()
   ▼
connecting ──(网络错误/建连超时)──▶ reconnecting ──(指数退避重试)──▶ connecting
   │  WS 打开，发送 LOGIN
   ▼
authenticating ──(Token 过期)──▶ 串行刷新 Token 后重连
   │  收到 LOGIN_ACK
   ▼
syncing ──(触发 SYNC_REQUEST 补齐离线消息)──▶ connected
   │                                              │
   └──(收到 KICK_DEVICE / logout / Refresh 失败)──┴──▶ kicked / disconnected
```

重连退避规则：
- 初始间隔 `1s`，倍增上限 `30s`，叠加 `±20%` 随机抖动防止服务端重连风暴。
- 通过 `NWPathMonitor` 监听网络从无到有切换，以及 App 从后台切回前台（`UIApplication.willEnterForegroundNotification`）时立即重置退避并触发一次快速重连 + 增量同步。

---

## 3. 本地 SQLite 存储模型

每个 `(app_id, user_id)` 使用独立的数据库文件（如 `Application Support/NextIM/{app_id}_{user_id}/im.sqlite`），启用 `WAL` 模式并配置 `FileProtectionType.completeUntilFirstUserAuthentication`；严禁把数据库放在易被系统清理的 `Caches` 目录。

### 3.1 `local_messages`
| 字段 | 类型 | 说明 |
|---|---|---|
| `client_msg_id` | `TEXT PRIMARY KEY` | 本地主键（发送前即生成，支持离线排队与发送中展示） |
| `message_id` | `INTEGER` | 服务端返回后回填，建唯一索引（非 0 时） |
| `conversation_id` | `TEXT NOT NULL` | 会话 ID |
| `seq` | `INTEGER NOT NULL DEFAULT 0` | 发送中为 `0`，Server Accepted 后写入服务端 `seq` |
| `sender_id` | `TEXT NOT NULL` | 发送者 |
| `message_type` | `INTEGER NOT NULL` | 对齐 Proto `MessageType` |
| `content` | `BLOB NOT NULL` | 消息载荷 |
| `local_state` | `INTEGER NOT NULL` | `1=sending, 2=sent(server_accepted), 3=delivered, 4=read, 5=failed, 6=recalled` |
| `server_time_ms` | `INTEGER NOT NULL` | 排序与展示时间（发送中暂用本地校准时间，ACK 后以服务端为准） |
| `retry_count` | `INTEGER NOT NULL DEFAULT 0` | 发送重试计数 |

关键索引：
- `UNIQUE INDEX uk_msg_id ON local_messages(message_id) WHERE message_id > 0`
- `UNIQUE INDEX uk_conv_seq ON local_messages(conversation_id, seq) WHERE seq > 0`
- `INDEX idx_conv_order ON local_messages(conversation_id, seq DESC, server_time_ms DESC)`

### 3.2 `local_conversations`
保存 `conversation_id`、`conversation_type`、`peer_id`、`last_seq`（已连续落库的本地同步游标）、`read_seq`、`unread_count`、`last_message_id`、`last_message_at_ms`、`is_pinned`、`is_muted`。

### 3.3 `sync_gaps`（序列缺口记录，可选但强烈推荐）
当收到实时推送 `seq = 12`，而当前连续游标 `last_seq = 10` 时：
- 消息 `seq = 12` 正常入库展示或暂存，但 **`last_seq` 严禁直接跳到 12**；
- 记录缺口 `[11, 11]` 并触发 `SYNC_REQUEST(last_seq = 10)`；
- 当 `seq = 11` 补齐落库后，再将连续游标 `last_seq` 推进到 `12`，并回发 `MESSAGE_DELIVERED_ACK(continuous_seq = 12)`。

---

## 4. 核心行为硬约束

1. **先落库再 ACK**：收到 `PUSH_MESSAGE` 时，必须在 SQLite 事务提交（写入 `local_messages` 并更新 `local_conversations`）**之后**才发送 `MESSAGE_DELIVERED_ACK`。
2. **发送幂等**：消息进入发送队列前先以 `local_state = sending` 写入 `local_messages`；重连或超时重试时复用同一个 `client_msg_id`，绝不生成新 ID。
3. **Token 串行刷新**：多个并发请求同时遇到 `10006 TOKEN_EXPIRED` 时，由 `AuthManager actor` 合并为单次 `/api/v1/auth/refresh` 请求，避免 Refresh Token 轮换触发误判重放攻击。
4. **凭证存储**：`access_token`、`refresh_token` 和 `device_id` 存放在 iOS Keychain（`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`），登出时彻底清除 Token（但保留稳定 `device_id`）。
