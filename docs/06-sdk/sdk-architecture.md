# 多端 SDK 统一架构

## 1. 公共能力

所有平台的 SDK 必须保持语义一致：
- initialize / login / logout
- 连接状态、心跳、重连
- 消息发送、接收、撤回
- 本地持久化和消息去重
- 会话列表、未读数
- 历史消息、离线同步、游标
- 好友和群组
- Delivered / Read 状态
- 文件上传下载
- 统一错误模型、日志和监听器

## 2. 内部逻辑层

```text
Public API
  |
IM Client
  |-- Auth Manager
  |-- Connection Manager
  |-- Message Manager
  |-- Conversation Manager
  |-- Sync Manager
  |-- Friend / Group Manager
  |-- File / Push Manager
  |
Local Database
  |
HTTP + WebSocket Transport
```

## 3. 平台技术

| 平台 | 方案 |
|---|---|
| iOS | Swift、URLSessionWebSocketTask（或评估后的库）、SQLite、Keychain |
| Android | Kotlin、Coroutines / Flow、OkHttp、Room、WorkManager |
| Flutter | Dart Plugin，首期桥接 iOS / Android |
| Web | TypeScript、WebSocket、IndexedDB |

## 4. 一致性要求

- 所有端共用同一协议定义。
- 所有端共用消息状态语义。
- 每个平台有独立测试，但共享一致性测试用例。
- SDK 公共 API 采用语义版本。
- 数据库 schema 迁移需兼容升级和降级策略。
- 连接状态、消息状态和同步状态不得由 UI 层自行推断。

## 5. iOS 特别要求

- 后台运行不保证 WebSocket 长连接持续；通过 APNs 和前台恢复后的增量同步补偿。
- 本地数据库先落库再 ACK。
- Keychain 保存凭证。
- 线程安全和 actor / queue 所有权清晰。
- 支持 Swift Package；必要时提供 XCFramework。
- 规划 Objective-C 调用兼容层（如有客户需求）。
