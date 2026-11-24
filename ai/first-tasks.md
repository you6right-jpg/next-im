# 首批 AI 开发任务顺序

请一次只执行一项任务，完成并审查后再开始下一项。

1. 检查仓库现状，生成工程结构建议；不写业务代码。
2. 建立 Go 工程骨架、配置加载、结构化日志、统一错误。
3. 建立 MySQL 连接、Migration runner、健康检查。
4. 建立 Redis 封装和测试。
5. 建立 tenant / app 上下文与 HTTP 中间件。
6. 定义 Protobuf 公共 Envelope、登录和心跳协议。
7. 生成 Go 协议代码并建立协议兼容测试。
8. 实现 App 认证、用户登录、Token 签发和刷新。
9. 实现 WebSocket Gateway 的连接生命周期和心跳。
10. 实现设备连接注册和 Redis 路由。
11. 设计并实现单聊会话模型与 seq 原子分配。
12. 实现消息幂等、事务持久化和 Server Accepted ACK。
13. 实现 Outbox Worker 和消息分发。
14. 实现接收端 Delivered ACK 与状态更新。
15. 实现离线同步和游标。
16. 编写端到端故障测试。
17. 开始 iOS SDK 网络、连接和本地存储层。

每个任务都必须引用相关文档，并按 `ai/task-template.md` 提交完成报告。
