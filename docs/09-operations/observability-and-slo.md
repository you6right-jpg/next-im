# 可观测性与服务目标

## 必备指标

- API 请求量、错误率、P50 / P95 / P99。
- WebSocket 在线连接数、建连失败率、断连率。
- 登录成功率和耗时。
- 消息 Server Accepted 延迟。
- 消息分发延迟。
- Delivered ACK 延迟。
- 同步请求耗时、同步积压。
- Outbox 未处理数量和最老事件年龄。
- Kafka consumer lag。
- MySQL 连接数、慢查询、锁等待。
- Redis 内存、命中率、延迟。
- Push / Webhook 成功率。
- 对象存储错误率和流量。

## 日志与追踪

每个请求至少包含 request_id / trace_id、tenant_id、app_id（注意脱敏）、user_id（按安全策略处理）。消息链路应可用 message_id 关联客户端、网关、业务服务和 Worker 日志。

## SLA 制定

性能目标必须绑定：
- 明确硬件和节点规格。
- 明确客户端 / 网络条件。
- 明确消息大小和业务比例。
- 明确在线用户与连接分布。
- 明确测试时长和错误率口径。

没有压测报告前，不对外承诺并发连接、TPS 或 P99 延迟。
