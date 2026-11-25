# Go 服务端模块开发模板

为了保持 `next-im-server` 中各个业务模块（如 Auth, Message, Conversation 等）的代码结构和设计风格一致，我们定义了以下标准的模块开发模板。

## 1. 目录结构

每个独立的业务领域模块（位于 `internal/` 下）应当遵循以下标准分层结构。以 `message` 模块为例：

```text
internal/message/
├── event/         # 领域事件定义（例如：消息已保存、消息已撤回）
├── handler/       # 接口适配层（HTTP API / WebSocket 请求的处理，参数校验）
├── model/         # 领域模型定义（实体、值对象及状态机）
├── repository/    # 存储接口定义及实现（MySQL / Redis 等的抽象）
├── service/       # 业务逻辑与事务边界编排
└── message.go     # 模块的入口/初始化文件（依赖注入、路由注册）
```

## 2. 各层职责与代码示例

### 2.1 Model 层 (`model/message.go`)
定义业务实体，不依赖任何存储实现框架。

```go
package model

import "time"

type Message struct {
    ID             string
    AppID          string
    ConversationID string
    SenderID       string
    ClientMsgID    string
    Seq            int64
    Type           int
    Content        string
    CreatedAt      time.Time
}
```

### 2.2 Repository 层 (`repository/message_repo.go`)
定义存储接口。实现类通常放在子目录或同级的文件中。

```go
package repository

import (
    "context"
    "next-im/internal/message/model"
)

type MessageRepository interface {
    Save(ctx context.Context, msg *model.Message) error
    FindBySeqRange(ctx context.Context, appID, convID string, startSeq, endSeq int64) ([]*model.Message, error)
}
```

### 2.3 Service 层 (`service/message_service.go`)
负责核心业务逻辑。所有跨层调用、事务编排、事件派发都在此层完成。

```go
package service

import (
    "context"
    "next-im/internal/message/model"
    "next-im/internal/message/repository"
)

type MessageService interface {
    SendMessage(ctx context.Context, req *SendRequest) (*model.Message, error)
}

type messageServiceImpl struct {
    repo repository.MessageRepository
    // ... 其他依赖，如发号器、事务管理器等
}

// SendRequest 是 Service 层的 DTO，不暴露底层 HTTP/WS 结构
type SendRequest struct {
    AppID       string
    SenderID    string
    ConvID      string
    ClientMsgID string
    Content     string
}

func (s *messageServiceImpl) SendMessage(ctx context.Context, req *SendRequest) (*model.Message, error) {
    // 1. 业务校验
    // 2. 幂等检查
    // 3. 生成 seq 和 ID
    // 4. 保存数据库及发送 Outbox 事件（在一个事务中）
    // ...
    return msg, nil
}
```

### 2.4 Handler 层 (`handler/http.go` 或 `handler/ws.go`)
解析入站请求，调用 Service，并封装统一的错误响应。**严禁在 Handler 中直接操作 Repository。**

```go
package handler

import (
    "net/http"
    // ... 框架包
)

type MessageHandler struct {
    svc service.MessageService
}

func (h *MessageHandler) HandleSend(w http.ResponseWriter, r *http.Request) {
    // 1. 从 context 获取租户信息 (Tenant/AppID)
    // 2. 解析请求 Body 到 DTO
    // 3. 调用 h.svc.SendMessage
    // 4. 返回标准 JSON 格式或 Error Code
}
```

## 3. 依赖注入与初始化
模块应当在入口文件中提供工厂函数，方便在 `cmd/` 中组装。

```go
// internal/message/message.go
package message

func NewModule(repo repository.MessageRepository) service.MessageService {
    return service.NewMessageService(repo)
}
```

## 4. 关键规范提醒
- **Context 传递**：所有跨层调用必须将 `context.Context` 作为第一个参数，用于传递请求级变量（TraceID, 租户信息）以及超时/取消控制。
- **错误处理**：使用统一的 `errors` 包包装底层错误，不要直接向外抛出数据库的具体错误（防数据泄露）。
