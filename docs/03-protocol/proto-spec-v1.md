# Protobuf 协议 v1 详细规范

本文档是 `next-im-protocol` 仓库及 Go / Swift / Kotlin / TypeScript 协议代码生成的权威契约（对应任务 6、7、9、12、14、15）。

## 1. 仓库目录与命名规范

```text
next-im-protocol/
├── buf.yaml
├── buf.gen.yaml
├── proto/
│   └── nextim/
│       └── protocol/
│           └── v1/
│               ├── envelope.proto     # Envelope 与 Command 枚举
│               ├── auth.proto         # LOGIN / LOGIN_ACK / KICK_DEVICE
│               ├── heartbeat.proto    # HEARTBEAT / HEARTBEAT_ACK
│               ├── message.proto      # Message 模型、SEND / ACK / PUSH / RECEIPT
│               ├── sync.proto         # SYNC_REQUEST / SYNC_RESPONSE / SYNC_FINISH
│               └── error.proto        # ErrorResponse
└── gen/
    ├── go/                            # 由 buf 生成，禁止手改
    ├── swift/
    ├── kotlin/
    └── ts/
```

包与选项约定：
- `syntax = "proto3";`
- `package nextim.protocol.v1;`
- `option go_package = "github.com/you6right-jpg/next-im-protocol/gen/go/nextim/protocol/v1;protocolv1";`
- 协议版本号：`protocol_version = 1`。

---

## 2. Command 号段分配（不可复用）

所有命令号一旦发布严禁修改或复用，废弃命令号必须 `reserved`。

| 号段 | 分类 | 枚举名 | 编号 | 方向 | 对应 Payload |
|---|---|---|---:|---|---|
| 0 | 未指定 | `COMMAND_UNSPECIFIED` | 0 | - | 禁止使用 |
| 100-199 | 连接与鉴权 | `COMMAND_LOGIN` | 101 | C -> S | `LoginRequest` |
| | | `COMMAND_LOGIN_ACK` | 102 | S -> C | `LoginResponse` |
| | | `COMMAND_KICK_DEVICE` | 103 | S -> C | `KickDeviceNotice` |
| 200-299 | 心跳保活 | `COMMAND_HEARTBEAT` | 201 | C -> S | `HeartbeatRequest` |
| | | `COMMAND_HEARTBEAT_ACK` | 202 | S -> C | `HeartbeatResponse` |
| 300-399 | 消息收发与回执 | `COMMAND_SEND_MESSAGE` | 301 | C -> S | `SendMessageRequest` |
| | | `COMMAND_SEND_MESSAGE_ACK` | 302 | S -> C | `SendMessageAck` |
| | | `COMMAND_PUSH_MESSAGE` | 303 | S -> C | `PushMessage` |
| | | `COMMAND_MESSAGE_DELIVERED_ACK` | 304 | C -> S | `MessageDeliveredAck` |
| | | `COMMAND_MESSAGE_READ_ACK` | 305 | C -> S | `MessageReadAck` |
| | | `COMMAND_PUSH_RECEIPT` | 306 | S -> C | `PushReceiptNotice` |
| 400-499 | 离线与增量同步 | `COMMAND_SYNC_REQUEST` | 401 | C -> S | `SyncRequest` |
| | | `COMMAND_SYNC_RESPONSE` | 402 | S -> C | `SyncResponse` |
| | | `COMMAND_SYNC_FINISH` | 403 | S -> C | `SyncFinish` |
| 500-599 | 通用错误与控制 | `COMMAND_ERROR` | 500 | S -> C | `ErrorResponse` |

---

## 3. 核心 Proto 定义

### 3.1 `envelope.proto`

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

enum Command {
  COMMAND_UNSPECIFIED = 0;

  COMMAND_LOGIN = 101;
  COMMAND_LOGIN_ACK = 102;
  COMMAND_KICK_DEVICE = 103;

  COMMAND_HEARTBEAT = 201;
  COMMAND_HEARTBEAT_ACK = 202;

  COMMAND_SEND_MESSAGE = 301;
  COMMAND_SEND_MESSAGE_ACK = 302;
  COMMAND_PUSH_MESSAGE = 303;
  COMMAND_MESSAGE_DELIVERED_ACK = 304;
  COMMAND_MESSAGE_READ_ACK = 305;
  COMMAND_PUSH_RECEIPT = 306;

  COMMAND_SYNC_REQUEST = 401;
  COMMAND_SYNC_RESPONSE = 402;
  COMMAND_SYNC_FINISH = 403;

  COMMAND_ERROR = 500;
}

message Envelope {
  uint32 protocol_version = 1; // 当前固定为 1
  Command command = 2;
  uint64 request_id = 3;       // 客户端请求关联 ID；服务端主动推送可为 0 或服务端流水号
  int64 timestamp_ms = 4;      // UTC 毫秒时间戳
  bytes payload = 5;           // 对应 command 的 Protobuf 序列化字节
}
```

### 3.2 `auth.proto`

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

enum Platform {
  PLATFORM_UNSPECIFIED = 0;
  PLATFORM_IOS = 1;
  PLATFORM_ANDROID = 2;
  PLATFORM_WEB = 3;
  PLATFORM_DESKTOP = 4;
  PLATFORM_FLUTTER_IOS = 5;
  PLATFORM_FLUTTER_ANDROID = 6;
}

message LoginRequest {
  string access_token = 1;
  string device_id = 2;
  Platform platform = 3;
  string sdk_version = 4;
  string app_version = 5;
  repeated string capabilities = 6; // 能力协商列表，如 ["sync_v1", "receipt_v1"]
}

message LoginResponse {
  string conn_id = 1;
  string user_id = 2;
  uint32 heartbeat_interval_sec = 3; // 服务端推荐心跳间隔，默认 30
  uint32 heartbeat_timeout_sec = 4;  // 服务端超时判定时长，默认 90
  int64 server_time_ms = 5;
  repeated string server_capabilities = 6;
}

enum KickReason {
  KICK_REASON_UNSPECIFIED = 0;
  KICK_REASON_SAME_PLATFORM_LOGIN = 1;
  KICK_REASON_ADMIN_REVOKED = 2;
  KICK_REASON_TOKEN_EXPIRED = 3;
  KICK_REASON_USER_DISABLED = 4;
}

message KickDeviceNotice {
  KickReason reason = 1;
  string message = 2;
  string triggered_by_device_id = 3;
}
```

### 3.3 `heartbeat.proto`

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

message HeartbeatRequest {
  int64 client_time_ms = 1;
}

message HeartbeatResponse {
  int64 server_time_ms = 1;
}
```

### 3.4 `message.proto`

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

enum ConversationType {
  CONVERSATION_TYPE_UNSPECIFIED = 0;
  CONVERSATION_TYPE_SINGLE = 1;
  CONVERSATION_TYPE_GROUP = 2;
}

enum MessageType {
  MESSAGE_TYPE_UNSPECIFIED = 0;
  MESSAGE_TYPE_TEXT = 1;
  MESSAGE_TYPE_IMAGE = 2;
  MESSAGE_TYPE_AUDIO = 3;
  MESSAGE_TYPE_VIDEO = 4;
  MESSAGE_TYPE_FILE = 5;
  MESSAGE_TYPE_LOCATION = 6;
  MESSAGE_TYPE_CARD = 7;
  MESSAGE_TYPE_CUSTOM = 100;
  MESSAGE_TYPE_SYSTEM = 200;
}

enum MessageStatus {
  MESSAGE_STATUS_UNSPECIFIED = 0;
  MESSAGE_STATUS_NORMAL = 1;
  MESSAGE_STATUS_RECALLED = 2;
  MESSAGE_STATUS_DELETED_BY_ADMIN = 3;
}

message MessageItem {
  uint64 message_id = 1;
  string client_msg_id = 2;
  string conversation_id = 3;
  ConversationType conversation_type = 4;
  uint64 seq = 5;
  string sender_id = 6;
  string receiver_id = 7;
  MessageType message_type = 8;
  bytes content = 9;           // UTF-8 JSON 或结构化负载
  MessageStatus status = 10;
  int64 server_time_ms = 11;
  int64 recalled_at_ms = 12;
}

message SendMessageRequest {
  string client_msg_id = 1;
  string conversation_id = 2;  // 单聊可由客户端按规则构造，也可留空由 receiver_id 派生并校验一致
  ConversationType conversation_type = 3;
  string receiver_id = 4;      // 单聊为对端 user_id，群聊为 group_id
  MessageType message_type = 5;
  bytes content = 6;
}

message SendMessageAck {
  string client_msg_id = 1;
  uint64 message_id = 2;
  string conversation_id = 3;
  uint64 seq = 4;
  int64 server_time_ms = 5;
  bool duplicate = 6;          // true 表示命中幂等，返回既有消息结果
}

message PushMessage {
  MessageItem message = 1;
}

message MessageDeliveredAck {
  string conversation_id = 1;
  repeated uint64 message_ids = 2;
  uint64 continuous_seq = 3;   // 可选：客户端已连续落库的最大 seq
}

message MessageReadAck {
  string conversation_id = 1;
  uint64 read_seq = 2;
}

message PushReceiptNotice {
  string conversation_id = 1;
  string user_id = 2;
  uint64 delivered_seq = 3;
  uint64 read_seq = 4;
  repeated uint64 delivered_message_ids = 5;
  int64 timestamp_ms = 6;
}
```

### 3.5 `sync.proto`

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

import "nextim/protocol/v1/message.proto";

message ConversationCursor {
  string conversation_id = 1;
  uint64 last_seq = 2;         // 客户端已连续落库的 seq，首次同步传 0
}

message SyncRequest {
  string device_id = 1;
  repeated ConversationCursor cursors = 2;
  uint32 limit_per_conversation = 3; // 默认 50，最大 200
}

message ConversationSyncChunk {
  string conversation_id = 1;
  repeated MessageItem messages = 2;
  uint64 max_seq = 3;          // 服务端该会话当前最新 seq
  bool has_more = 4;
  bool cursor_reset_required = 5; // 游标失效/超出保留期时为 true，客户端需重建基线
}

message SyncResponse {
  repeated ConversationSyncChunk chunks = 1;
  int64 server_time_ms = 2;
}

message SyncFinish {
  int64 server_time_ms = 1;
}
```

### 3.6 `error.proto`

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

import "nextim/protocol/v1/envelope.proto";

message ErrorResponse {
  uint32 code = 1;             // 对齐 docs/03-protocol/error-codes.md
  string message = 2;          // 安全的可读说明，不暴露内部堆栈
  Command ref_command = 3;     // 引发错误的原始命令
  bool retryable = 4;          // 客户端是否允许按退避策略重试
  uint32 retry_after_ms = 5;   // 限流或过载时建议等待毫秒数
}
```

---

## 4. 协议兼容测试要求（对应任务 7）

- 维护黄金二进制样本（Golden Fixtures）：将 v1 各核心消息序列化字节保存为测试样本，确保每次协议演进后反序列化仍完全兼容。
- CI 中接入 `buf lint` 与 `buf breaking --against '.git#branch=main'`，阻止破坏性字段修改。
- 单条帧大小限制：WebSocket Gateway 强制限制单帧最大 `64 KB`（媒体文件严禁内联，须走对象存储上传接口）。
