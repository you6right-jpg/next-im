# 通信协议总则

## 1. 通信选型

| 场景 | 通道 |
|---|---|
| 登录、资料、好友、群组、会话管理 | HTTPS REST |
| 实时消息、心跳、ACK、在线事件 | WSS |
| 文件上传 | HTTPS 直传对象存储或服务端中转 |
| 文件下载 | 带权限校验的签名 URL / CDN |
| 服务间调用 | 模块接口；需要跨进程时使用 gRPC |
| 数据结构 | Protobuf 为协议权威定义 |

## 2. WebSocket Envelope

```protobuf
syntax = "proto3";
package nextim.protocol.v1;

message Envelope {
  uint32 protocol_version = 1;
  uint32 command = 2;
  uint64 request_id = 3;
  int64 timestamp_ms = 4;
  bytes payload = 5;
}
```

每个 command 对应独立请求 / 响应消息。命令号一旦发布不得复用。

## 3. 命令分类

- CONNECT / LOGIN / LOGIN_ACK
- HEARTBEAT / HEARTBEAT_ACK
- SEND_MESSAGE / SEND_MESSAGE_ACK
- PUSH_MESSAGE
- MESSAGE_DELIVERED_ACK
- MESSAGE_READ_ACK
- SYNC_REQUEST / SYNC_RESPONSE / SYNC_FINISH
- KICK_DEVICE
- ERROR

## 4. 协议演进

- 字段只新增，不随意改变既有字段类型和含义。
- 删除字段时保留 field number，使用 `reserved`。
- enum 数值不得重排或复用。
- 新客户端能力通过 capability 协商。
- 不兼容变化升级协议主版本。
- 生成代码必须由统一脚本生成，不手改 generated 文件。
- 维护协议变更日志和跨语言兼容测试。
