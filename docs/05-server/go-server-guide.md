# Go 服务端开发规范

## 1. 推荐目录

```text
im-server/
├── cmd/
│   ├── im-api/
│   ├── im-gateway/
│   ├── im-worker/
│   └── im-migrate/
├── internal/
│   ├── app/
│   ├── auth/
│   ├── tenant/
│   ├── user/
│   ├── device/
│   ├── connection/
│   ├── message/
│   ├── conversation/
│   ├── sync/
│   ├── friend/
│   ├── group/
│   ├── receipt/
│   ├── file/
│   ├── push/
│   ├── search/
│   ├── webhook/
│   ├── admin/
│   ├── billing/
│   └── license/
├── pkg/
│   ├── protocol/
│   ├── database/
│   ├── redis/
│   ├── kafka/
│   ├── logger/
│   ├── errors/
│   ├── idgen/
│   └── observability/
├── migrations/
├── configs/
├── deployments/
├── scripts/
└── tests/
```

## 2. 模块分层

每个模块可采用：

```text
message/
  handler/
  service/
  repository/
  model/
  event/
```

- Handler：协议适配、参数校验、身份上下文。
- Service：业务用例、事务边界、领域规则。
- Repository：存储抽象及实现。
- Model：领域模型和状态。
- Event：领域事件定义。

## 3. 工程规范

- 使用 `context.Context` 贯穿请求和异步任务。
- 明确超时、取消和重试策略。
- 错误使用 wrapping，保留内部 cause。
- 日志结构化，必须有 request_id / trace_id。
- 不记录 Token、Secret 和敏感消息正文。
- 配置外置，支持环境区分。
- 数据库事务由业务 Service 明确管理。
- Repository 接口由使用方或领域模块定义。
- 并发共享状态必须有明确同步策略。
- 所有后台 goroutine 必须有生命周期和退出机制。

## 4. 不允许

- Handler 直接操作数据库。
- 跨模块随意读写其他模块表。
- 忽略错误或吞掉异常。
- 通过全局可变变量保存请求态。
- 将密钥写入仓库。
- 在没有压测和 ADR 的情况下引入复杂中间件。
