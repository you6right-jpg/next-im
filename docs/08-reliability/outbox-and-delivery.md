# Transactional Outbox 与消息分发设计

本文档细化 `message-reliability.md`，明确 Outbox 事件契约、Worker 租约领取机制、重放幂等与多设备分发流程（对应任务 12、13、14、16）。

## 1. Outbox 事件契约

所有核心领域状态变更都在 MySQL 业务事务内写入 `im_outbox`，事件使用统一信封结构（存入 `payload_json`）：

```json
{
  "event_id": "01jf8m01aabbccdd",
  "event_type": "message.created",
  "schema_version": 1,
  "tenant_id": "tnt_01jf8k...",
  "app_id": "app_01jf8k...",
  "occurred_at_ms": 1732500000123,
  "data": {
    "message_id": "728391048291041281",
    "client_msg_id": "01jf8m0099aa11bb",
    "conversation_id": "c2c:usr_10001:usr_10002",
    "conversation_type": 1,
    "seq": 42,
    "sender_id": "usr_10001",
    "receiver_id": "usr_10002",
    "message_type": 1,
    "content": "{\"text\":\"hello\"}",
    "server_time_ms": 1732500000123
  }
}
```

### P0/P1 核心事件类型清单

| `event_type` | `aggregate_type` | `aggregate_id` | 消费者动作 |
|---|---|---|---|
| `message.created` | `message` | `conversation_id` | 路由查询目标用户及发送方其他设备，实时推送 `PUSH_MESSAGE`；离线设备触发推送任务（P1） |
| `message.recalled` | `message` | `conversation_id` | 通知会话相关在线设备更新消息撤回状态 |
| `receipt.delivered` | `receipt` | `conversation_id` | 向消息发送方在线设备推送 `PUSH_RECEIPT` |
| `receipt.read` | `receipt` | `conversation_id` | 通知发送方已读状态，并同步发送方/接收方多端未读数清零 |
| `device.kicked` | `device` | `user_id` | 通知目标 Gateway 节点发送 `KICK_DEVICE` 并断开旧连接 |

---

## 2. Outbox Worker 状态机与并发领取

### 2.1 状态流转
`1 (pending)` -> `2 (processing)` -> `3 (published)`
失败时：`2 (processing)` -> `1 (pending)`（`retry_count + 1`，按指数退避设置 `next_retry_at`）；超过 `max_retries`（默认 10）进入 `4 (dead_letter)` 并触发告警。

### 2.2 多实例安全领取（MySQL 8 `SKIP LOCKED`）
为支持多 Worker 实例水平扩展而不重复抢占同一批记录，领取采用短事务 + 行租约模式：

```sql
-- 1. 选出待处理或租约已过期的事件（防止某 Worker 宕机后事件永久卡在 processing）
SELECT id, event_id, event_type, aggregate_id, payload_json, retry_count
FROM im_outbox
WHERE (status = 1 AND next_retry_at <= NOW(3))
   OR (status = 2 AND locked_until < NOW(3))
ORDER BY id ASC
LIMIT ?
FOR UPDATE SKIP LOCKED;

-- 2. 在同一短事务内将选中行置为 processing 并设置租约（如 NOW(3) + 30s）
UPDATE im_outbox
SET status = 2, locked_by = ?, locked_until = ?, updated_at = NOW(3)
WHERE id IN (...);
```

### 2.3 指数退避策略
- 第 `n` 次重试等待时间：`min(base_delay * 2^(n-1), max_delay) + random_jitter`，例如 `base_delay = 1s`，`max_delay = 60s`。
- 单条事件处理完成后立即更新 `status = 3`；若 Worker 在外部投递成功后、更新 DB 前宕机，租约过期后事件会被重新投递（At-least-once），下游分发器与接收端 SDK 必须保持幂等。

---

## 3. 消息分发抽象 (`EventPublisher` & `MessageDispatcher`)

为满足 `ADR-0001`（P0 不被 Kafka 阻塞、规模化平滑接入 Kafka），Worker 侧分层定义两个接口：

1. **`EventPublisher`（发布侧抽象）**
   - P0 默认实现：`DirectDispatchPublisher`（Outbox Worker 直接调用本地或 Redis Pub/Sub `MessageDispatcher` 完成分发）。
   - 规模化实现：`KafkaEventPublisher`（按 `aggregate_id = conversation_id` 作为 Kafka Partition Key 保证同会话顺序，投递到 Topic `nextim.events.message`）。
2. **`MessageDispatcher`（在线投递执行）**
   处理 `message.created` 时：
   1. 确定目标接收用户集合：
      - 单聊：`receiver_id` 的所有在线设备 + `sender_id` 除发送设备外的其他在线设备（多端漫游同步）。
      - 群聊（P1）：群成员列表（小群读扩散/写扩散按群规模分流，大群在线拉取同步）。
   2. 批量查询 Redis `nextim:{tid}:{aid}:conn:user:{uid}`，过滤掉心跳已超时的过期路由条目。
   3. 按目标 `gateway_id` 分组聚合，将 `PushMessage` 投递给对应 Gateway 节点（同进程直调或通过 `nextim:sys:gw:push:{gateway_id}` / gRPC 推送）。
   4. 对没有在线设备的目标用户（或长时间未返回 `MESSAGE_DELIVERED_ACK`），记入离线状态，等待其重连后走 `SYNC_REQUEST` 补齐（P1 阶段异步触发 APNs/FCM 离线提醒）。

---

## 4. Delivered ACK 与重发边界

- Gateway 将 `PUSH_MESSAGE` 写入 WebSocket 后，并不直接标记 `Delivered`。
- 接收端 SDK 在本地数据库完成事务落库后，回发 `COMMAND_MESSAGE_DELIVERED_ACK`。
- 服务端收到 `MESSAGE_DELIVERED_ACK` 后：
  1. 校验该用户确为该会话成员。
  2. 幂等更新 `im_message_user_state.delivered_at`（单聊）以及 `im_user_sync.last_seq`（仅单调向前推进：`GREATEST(last_seq, continuous_seq)`）。
  3. 若需要向发送方展示“已送达”回执，写一条 `receipt.delivered` 轻量通知或异步推送 `PUSH_RECEIPT`。
- **为什么不在服务端无限重推单条 WS 帧**：重连或弱网丢包时，相比在 Gateway 内存堆积重发定时器，客户端重连后按 `last_seq` 发起 `SYNC_REQUEST` 是更可靠且无状态的唯一恢复真相源。
