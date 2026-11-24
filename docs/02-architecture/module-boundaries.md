# 模块边界与职责

| 模块 | 负责 | 不负责 |
|---|---|---|
| Auth | 身份校验、Token、应用凭证 | 消息业务 |
| Tenant / App | 租户、应用配置和隔离上下文 | 终端消息内容处理 |
| User | 用户资料、状态 | 连接生命周期 |
| Device | 设备注册、设备状态 | WebSocket 传输 |
| Connection | 连接、心跳、节点路由 | 消息持久化规则 |
| Message | 消息校验、幂等、序列、持久化 | UI 会话展示 |
| Conversation | 会话元数据、用户会话状态 | 消息网络传输 |
| Sync | 游标、增量同步、历史拉取 | 推送通道 |
| Friend | 好友关系、申请、黑名单 | 消息存储 |
| Group | 群、成员、角色、群事件 | 长连接 |
| Receipt | Delivered / Read 状态 | APNs 推送 |
| File | 文件元数据、上传下载授权 | 文件内容长期存于 MySQL |
| Push | 推送任务和通道适配 | 可靠消息存储 |
| Search | 搜索索引和查询 | 作为消息事实数据源 |
| Webhook | 事件投递、签名、重试 | 核心事务同步等待回调 |
| Admin | 管理操作、审计 | 绕过领域服务直接改表 |
| Billing | 套餐、用量、账单 | 阻塞每条消息核心链路 |
| License | 私有化授权 | 影响消息协议语义 |

## Go 模块分层

每个模块建议：

```text
<module>/
  handler/      # HTTP / WS 入站适配
  service/      # 业务用例和事务编排
  repository/   # 存储接口及实现
  model/        # 领域模型、状态
  event/        # 领域事件
```

约束：
- Handler 不直接操作数据库。
- Service 不依赖具体 HTTP 框架。
- Repository 不承载业务规则。
- 跨模块通过接口或事件，不直接改别的模块的数据。
- 事务边界由拥有数据的领域服务定义。
