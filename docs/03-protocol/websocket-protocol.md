# WebSocket 协议与交互规范

## 1. 协议概述

WebSocket 通道是 Next-IM 的核心实时链路，主要负责客户端与服务端的双向实时数据传输。它承担以下职责：
- 连接的建立与鉴权（Login）
- 链路保活（Heartbeat）
- 实时消息的上下行（上行发消息，下行收推送）
- 消息的确认（ACK，包括 Server Accepted 和 Delivered）
- （可选）增量离线同步（Sync）

所有 WebSocket 数据帧必须使用 Protobuf 序列化后的二进制数据（Binary Frame）。

## 2. 基础信封 (Envelope) 设计

如 `protocol-overview.md` 中所定义，所有的 WS 消息包外层必须包裹统一的 `Envelope`。

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

message Envelope {
  // 协议大版本，当前为 1
  uint32 protocol_version = 1;
  // 命令字，用于区分具体的 payload 类型
  uint32 command = 2;
  // 客户端生成的请求序列号，用于将服务端的 ACK 关联到具体请求
  uint64 request_id = 3;
  // 消息生成时间戳
  int64 timestamp_ms = 4;
  // 具体的业务 Protobuf 序列化后的数据
  bytes payload = 5;
}
```

## 3. 核心交互流程

### 3.1 登录鉴权 (LOGIN)
建立 WS 连接后，客户端必须在规定的超时时间（如 5 秒）内发送 `LOGIN` 命令。未完成鉴权的连接将被服务端主动断开。

- **Client -> Server (LOGIN)**: 携带由 HTTP 接口获取的 Connection Token 或 Access Token、设备 ID、设备类型等信息。
- **Server -> Client (LOGIN_ACK)**: 鉴权成功，下发服务端的会话参数（如心跳间隔推荐值）。若失败则下发 `ERROR` 并断开。

### 3.2 心跳保活 (HEARTBEAT)
- 客户端按约定间隔发送 `HEARTBEAT` 报文。
- 服务端收到后必须回复 `HEARTBEAT_ACK`。
- 如果服务端连续 N 个周期未收到心跳，将判定设备离线，清理连接状态。
- （服务端不主动发起心跳请求，以降低服务端定时器开销）。

### 3.3 消息发送与确认 (上行)
客户端发送消息时，要求服务端必须给予明确的持久化成功确认。
- **Client -> Server (SEND_MESSAGE)**: 携带 `client_msg_id`。
- **Server -> Client (SEND_MESSAGE_ACK)**: 携带服务端生成的 `message_id`、`seq` 和时间戳。
  - *重试策略*：若客户端超时未收到 ACK，应复用原 `client_msg_id` 重新发送，服务端返回成功但不会产生两条消息。

### 3.4 实时消息下发与确认 (下行)
当有新消息产生时，服务端通过 WS 将消息推送给接收端。
- **Server -> Client (PUSH_MESSAGE)**: 包含完整的消息体及 `seq`。
- **Client -> Server (MESSAGE_DELIVERED_ACK)**: 客户端在完成本地数据库持久化后，必须回复送达确认，携带收到消息的 `message_id` 或最大的连续 `seq`。
  - *重试策略*：若服务端超时未收到 Delivered ACK，可能会重推，客户端必须根据 `message_id` 和 `seq` 做好去重。

### 3.5 被踢下线 (KICK_DEVICE)
当同一用户的同一类型设备在其他节点登录，或者管理员在后台主动封禁设备时，服务端下发该指令。
- **Server -> Client (KICK_DEVICE)**: 携带踢出的原因（如：被另一台设备顶替）。
- 客户端收到后应主动断开连接并清空内存态，弹出提示。

## 4. 异常处理 (ERROR)
当客户端的请求出现业务错误（如发送消息到不存在的群、Token过期等），服务端将返回通用的错误命令。
```protobuf
message ErrorResponse {
  uint32 code = 1;          // 参考 error-codes.md
  string message = 2;       // 调试信息
  uint32 ref_command = 3;   // 引发错误的原始 command
}
```
客户端根据 `request_id` 和 `ref_command` 将错误回调给上层业务。
