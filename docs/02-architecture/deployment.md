# 部署架构

## SaaS 参考拓扑

```text
Internet
  |
DNS / CDN / WAF / Load Balancer
  |--------------------------|
HTTP API                  WebSocket Gateway
  |                          |
API replicas              WS replicas
  |                          |
  +----------- Business -----+
                 |
       +---------+---------+
       |         |         |
     MySQL      Redis    Outbox/Kafka
                           |
                         Workers
                           |
                Push / Webhook / Search
                           |
                    S3-compatible storage
```

## 私有化拓扑

客户环境部署相同服务镜像和协议，可按规模选择：
- 单机 Docker Compose：开发、演示、小规模试用。
- 多节点 Docker / Kubernetes：生产环境。
- 独立数据库和对象存储：严格隔离客户。
- License 服务：在线或离线授权校验。

## 部署约束

- 所有服务无状态优先，连接状态由 Redis / 路由层协调。
- WebSocket Gateway 需要优雅下线和连接迁移策略。
- 数据库升级采用向前兼容的 expand-contract 方式。
- 配置与密钥通过环境变量或 Secret 管理。
- 备份必须验证恢复，不以“备份任务成功”作为可恢复证明。
