# 总体系统架构

## 1. 架构原则

1. 模块化单体起步，边界清晰，按负载和团队规模拆分服务。
2. HTTP 负责普通 API；WebSocket 负责实时双向通信。
3. Protobuf 是实时协议和跨端数据模型的权威定义。
4. MySQL 保存业务事实；Redis 负责缓存、在线状态和路由。
5. 消息持久化事务与事件发布通过 Outbox 衔接。
6. 所有租户数据访问经过统一上下文和授权层。
7. SaaS 与私有化使用同一核心代码和配置抽象。

## 2. 逻辑分层

- 客户端：iOS / Android / Flutter / Web / 服务端集成方。
- 接入层：HTTP API、WebSocket Gateway、鉴权、限流。
- 业务层：Auth、User、Device、Connection、Message、Conversation、Sync、Friend、Group、Receipt、File、Push、Search、Webhook、Admin、Billing、License。
- 数据层：MySQL、Redis、对象存储、OpenSearch。
- 异步层：Outbox、Kafka、Worker。
- 运维层：配置、日志、指标、链路追踪、告警、备份。

## 3. 逻辑拓扑

```text
Clients
  |-- HTTPS --> API Gateway --> Business Modules
  |-- WSS ----> WS Gateway ---> Connection / Message Routing
                                   |
                          Auth / Message / Sync
                                   |
                 +-----------------+----------------+
                 |                 |                |
               MySQL             Redis            Outbox
                                                    |
                                                   Kafka
                                                    |
                                      Push / Webhook / Workers
                                                    |
                                      APNs / FCM / Object Storage
```

## 4. 服务边界

### API Gateway
负责 HTTP 路由、鉴权中间件、限流、请求 ID、统一响应；不承载领域业务。

### WebSocket Gateway
负责连接生命周期、协议解码、心跳、连接认证、连接节点注册和上下行传输；不直接实现好友或群组领域规则。

### 业务模块
拥有各自领域模型和 Repository 接口。跨模块通过公开 Service 接口或领域事件协作，不允许随意读写其他模块内部表。

### Worker
执行 Outbox 投递、推送、Webhook、媒体处理和异步清理任务。所有任务消费者必须考虑重复执行。

## 5. 推荐部署单元

初期：
- `im-api`
- `im-gateway`
- `im-worker`
- MySQL
- Redis
- S3 / MinIO

规模化后按瓶颈拆分：
- API 服务
- WS Gateway 集群
- Message Service
- Sync Service
- Push / Webhook Worker
- Search Service

Kafka、OpenSearch、Kubernetes 可按实际阶段引入，不应成为最早期开发的阻塞项。
