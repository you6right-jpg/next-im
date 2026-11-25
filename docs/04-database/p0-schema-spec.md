# P0 核心数据库表结构详细设计（MySQL 8）

本文档定义了 P0 阶段（对应 `ai/first-tasks.md` 任务 3 ~ 16）涉及的所有 MySQL 8 表结构、枚举语义、约束索引与核心事务 SQL 模式。所有 DDL 变更必须通过 `migrations/` 下的版本化 SQL 文件执行，禁止手工改库。

## 1. 通用约定

- **字符集与排序规则**：`DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci`。
- **存储引擎**：`InnoDB`。
- **时间字段**：统一使用 `DATETIME(3)` 存储 UTC 时间（毫秒精度），避免 `TIMESTAMP` 2038 年上限问题。
- **隔离字段**：除平台级 `im_tenant` 表外，所有业务表必须包含 `tenant_id` 与 `app_id`。依据 `ADR-0002`，`app_id` 全局唯一，业务索引最左前缀统一使用 `app_id`，同时查询语句必须带入可信上下文的 `tenant_id` 与 `app_id`。
- **删除策略**：逐表明确软删除（`deleted_at` 或 `status`）或保留清理策略。

---

## 2. 平台与身份域

### 2.1 `im_tenant`（租户表）
- 删除策略：软删除（`status = 3` 或 `deleted_at`），禁止物理删除有业务数据的租户。

```sql
CREATE TABLE im_tenant (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id   VARCHAR(32)     NOT NULL COMMENT '全局唯一租户 ID, 如 tnt_xxx',
  name        VARCHAR(128)    NOT NULL COMMENT '租户名称',
  status      TINYINT         NOT NULL DEFAULT 1 COMMENT '1=active, 2=disabled, 3=deleted',
  created_at  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  deleted_at  DATETIME(3)     NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_tenant_id (tenant_id),
  KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 2.2 `im_app`（应用表）
- 删除策略：软删除（`status = 3`）。`app_secret_hash` 仅存单向哈希或 KMS 信封密文，绝不明文落库。

```sql
CREATE TABLE im_app (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       VARCHAR(32)     NOT NULL,
  app_id          VARCHAR(32)     NOT NULL COMMENT '全局唯一应用 ID, 如 app_xxx',
  name            VARCHAR(128)    NOT NULL,
  app_secret_hash VARCHAR(255)    NOT NULL COMMENT '服务端接口鉴权密钥哈希/信封密文',
  status          TINYINT         NOT NULL DEFAULT 1 COMMENT '1=active, 2=disabled, 3=deleted',
  config_json     JSON            NULL COMMENT '应用级配置（多设备策略、限流阈值等）',
  created_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  deleted_at      DATETIME(3)     NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_id (app_id),
  KEY idx_tenant_app (tenant_id, app_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 2.3 `im_user`（用户表）
- 删除策略：软删除（`status = 3`, `deleted_at`）。

```sql
CREATE TABLE im_user (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id   VARCHAR(32)     NOT NULL,
  app_id      VARCHAR(32)     NOT NULL,
  user_id     VARCHAR(64)     NOT NULL COMMENT '应用内唯一用户 ID',
  nickname    VARCHAR(128)    NOT NULL DEFAULT '',
  avatar_url  VARCHAR(512)    NOT NULL DEFAULT '',
  status      TINYINT         NOT NULL DEFAULT 1 COMMENT '1=active, 2=disabled, 3=deleted',
  extra_json  JSON            NULL,
  created_at  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  deleted_at  DATETIME(3)     NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_user (app_id, user_id),
  KEY idx_tenant_app_status (tenant_id, app_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 2.4 `im_device`（设备表）
- 删除策略：设备注销更新为 `status = 2 (revoked)`，历史设备记录按保留周期归档。

```sql
CREATE TABLE im_device (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       VARCHAR(32)     NOT NULL,
  app_id          VARCHAR(32)     NOT NULL,
  user_id         VARCHAR(64)     NOT NULL,
  device_id       VARCHAR(64)     NOT NULL COMMENT '客户端持久化设备 ID',
  platform        TINYINT         NOT NULL COMMENT '1=iOS, 2=Android, 3=Web, 4=Desktop, 5=Flutter_iOS, 6=Flutter_Android',
  device_name     VARCHAR(128)    NOT NULL DEFAULT '',
  app_version     VARCHAR(32)     NOT NULL DEFAULT '',
  sdk_version     VARCHAR(32)     NOT NULL DEFAULT '',
  push_token      VARCHAR(512)    NOT NULL DEFAULT '',
  push_provider   TINYINT         NOT NULL DEFAULT 0 COMMENT '0=none, 1=APNs, 2=FCM',
  status          TINYINT         NOT NULL DEFAULT 1 COMMENT '1=active, 2=revoked, 3=kicked',
  last_active_at  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  created_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_user_device (app_id, user_id, device_id),
  KEY idx_app_user_status (app_id, user_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 2.5 `im_auth_session`（Refresh Token 会话表）
- 用途：支持 Refresh Token 轮换（Rotation）、重放检测与按设备精确撤销。
- 删除策略：过期 30 天后由定时任务物理清理。

```sql
CREATE TABLE im_auth_session (
  id                  BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id           VARCHAR(32)     NOT NULL,
  app_id              VARCHAR(32)     NOT NULL,
  user_id             VARCHAR(64)     NOT NULL,
  device_id           VARCHAR(64)     NOT NULL,
  session_id          VARCHAR(64)     NOT NULL COMMENT '会话唯一 ID (jti/sid)',
  refresh_token_hash  CHAR(64)        NOT NULL COMMENT 'SHA-256(refresh_token)',
  expires_at          DATETIME(3)     NOT NULL,
  revoked_at          DATETIME(3)     NULL,
  created_at          DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at          DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_session_id (session_id),
  UNIQUE KEY uk_refresh_token_hash (refresh_token_hash),
  KEY idx_app_user_device (app_id, user_id, device_id),
  KEY idx_expires_at (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

---

## 3. 会话、消息与同步域

### 3.1 `im_conversation`（会话定义与原子序列表）
- 设计要点：将 `last_seq` 放在 `im_conversation` 行中，利用 InnoDB 行锁（`SELECT ... FOR UPDATE` 或原子 `UPDATE ... SET last_seq = LAST_INSERT_ID(last_seq + 1)`）在消息写入同一事务内保证 `seq` 严格单调递增、无跳号竞态。
- 删除策略：不物理删除会话主记录（保证消息历史与 `seq` 连续性），用户侧删除由 `im_conversation_user` 控制。

```sql
CREATE TABLE im_conversation (
  id                BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id         VARCHAR(32)     NOT NULL,
  app_id            VARCHAR(32)     NOT NULL,
  conversation_id   VARCHAR(160)    NOT NULL COMMENT 'c2c:{min_uid}:{max_uid} 或 grp:{group_id}',
  conversation_type TINYINT         NOT NULL COMMENT '1=single(c2c), 2=group',
  last_seq          BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '当前会话已分配的最大 seq',
  last_message_id   BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '最新一条消息的 message_id',
  last_message_at   DATETIME(3)     NULL,
  created_at        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_conv (app_id, conversation_id),
  KEY idx_tenant_app (tenant_id, app_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 3.2 `im_conversation_user`（用户维度会话状态表）
- 用途：驱动用户的会话列表排序、未读数、已读游标、置顶/免打扰与单侧删除（清除会话）。
- 删除策略：`is_deleted = 1` 且记录 `clear_before_seq`（用户再次收到新消息时恢复显示，但拉历史时过滤 `seq <= clear_before_seq`）。

```sql
CREATE TABLE im_conversation_user (
  id                BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id         VARCHAR(32)     NOT NULL,
  app_id            VARCHAR(32)     NOT NULL,
  user_id           VARCHAR(64)     NOT NULL,
  conversation_id   VARCHAR(160)    NOT NULL,
  conversation_type TINYINT         NOT NULL COMMENT '1=single, 2=group',
  peer_id           VARCHAR(64)     NOT NULL DEFAULT '' COMMENT '单聊对端 user_id 或群聊 group_id',
  read_seq          BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户已读到的最大连续 seq',
  delivered_seq     BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户已送达的最大连续 seq',
  clear_before_seq  BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '单侧清空历史水位线',
  unread_count      INT UNSIGNED    NOT NULL DEFAULT 0,
  is_pinned         TINYINT(1)      NOT NULL DEFAULT 0,
  is_muted          TINYINT(1)      NOT NULL DEFAULT 0,
  is_deleted        TINYINT(1)      NOT NULL DEFAULT 0,
  last_message_at   DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  created_at        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_user_conv (app_id, user_id, conversation_id),
  KEY idx_app_user_list (app_id, user_id, is_deleted, is_pinned, last_message_at DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 3.3 `im_message`（消息主表）
- 删除策略：撤回更新 `status = 2` 及 `recalled_at`（保留 `seq` 占位，防止同步出现序列空洞）；冷数据按归档策略整批迁移。

```sql
CREATE TABLE im_message (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       VARCHAR(32)     NOT NULL,
  app_id          VARCHAR(32)     NOT NULL,
  message_id      BIGINT UNSIGNED NOT NULL COMMENT 'Snowflake 全局唯一消息 ID',
  client_msg_id   VARCHAR(64)     NOT NULL COMMENT '客户端幂等 ID',
  conversation_id VARCHAR(160)    NOT NULL,
  seq             BIGINT UNSIGNED NOT NULL COMMENT '会话内单调严格递增序号',
  sender_id       VARCHAR(64)     NOT NULL,
  receiver_id     VARCHAR(64)     NOT NULL DEFAULT '' COMMENT '单聊对端 user_id，群聊可存 group_id',
  message_type    SMALLINT        NOT NULL COMMENT '1=text, 2=image, 3=audio, 4=video, 5=file, 6=location, 7=card, 100=custom, 200=system',
  content         MEDIUMTEXT      NOT NULL COMMENT 'JSON 或结构化文本载荷（不含大文件二进制）',
  status          TINYINT         NOT NULL DEFAULT 1 COMMENT '1=normal, 2=recalled, 3=deleted_by_admin',
  recalled_at     DATETIME(3)     NULL,
  created_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_message_id (app_id, message_id),
  UNIQUE KEY uk_app_sender_client_msg (app_id, sender_id, client_msg_id),
  UNIQUE KEY uk_app_conv_seq (app_id, conversation_id, seq),
  KEY idx_tenant_app_created (tenant_id, app_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 3.4 `im_message_user_state`（用户维度消息送达/已读状态表）
- 用途：精确追踪单聊消息（及小群开启回执时的）消息级 Delivered / Read 状态。大群默认采用会话级 `read_seq` 而非全量写此表。

```sql
CREATE TABLE im_message_user_state (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       VARCHAR(32)     NOT NULL,
  app_id          VARCHAR(32)     NOT NULL,
  message_id      BIGINT UNSIGNED NOT NULL,
  conversation_id VARCHAR(160)    NOT NULL,
  user_id         VARCHAR(64)     NOT NULL COMMENT '接收方 user_id',
  delivered_at    DATETIME(3)     NULL,
  read_at         DATETIME(3)     NULL,
  created_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_msg_user (app_id, message_id, user_id),
  KEY idx_app_conv_user (app_id, conversation_id, user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 3.5 `im_user_sync`（设备维度会话同步游标表）
- 用途：服务端记录各设备在各会话的连续确认同步位置，辅助离线推送判定、多设备进度对齐与游标诊断。

```sql
CREATE TABLE im_user_sync (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       VARCHAR(32)     NOT NULL,
  app_id          VARCHAR(32)     NOT NULL,
  user_id         VARCHAR(64)     NOT NULL,
  device_id       VARCHAR(64)     NOT NULL,
  conversation_id VARCHAR(160)    NOT NULL,
  last_seq        BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '该设备连续持久化并确认的最大 seq',
  updated_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_user_dev_conv (app_id, user_id, device_id, conversation_id),
  KEY idx_app_user_dev (app_id, user_id, device_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

---

## 4. 异步可靠投递域

### 4.1 `im_outbox`（事务发件箱表）
- 用途：与消息写入在同一 MySQL 事务中提交，保证“消息持久化”与“事件发布”的原子性。
- 删除策略：`status = 3 (published)` 且超过 7 天由归档任务清理；`status = 4 (dead_letter)` 保留并触发告警人工介入。

```sql
CREATE TABLE im_outbox (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  event_id        VARCHAR(36)     NOT NULL COMMENT 'ULID / UUIDv7 全局唯一事件 ID',
  tenant_id       VARCHAR(32)     NOT NULL,
  app_id          VARCHAR(32)     NOT NULL,
  aggregate_type  VARCHAR(32)     NOT NULL COMMENT '如 message, receipt, conversation',
  aggregate_id    VARCHAR(160)    NOT NULL COMMENT '如 conversation_id（作为分片/分区键）',
  event_type      VARCHAR(64)     NOT NULL COMMENT '如 message.created, message.recalled, receipt.delivered',
  payload_json    MEDIUMTEXT      NOT NULL COMMENT '事件 JSON 载荷',
  status          TINYINT         NOT NULL DEFAULT 1 COMMENT '1=pending, 2=processing, 3=published, 4=dead_letter',
  retry_count     INT UNSIGNED    NOT NULL DEFAULT 0,
  next_retry_at   DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  locked_by       VARCHAR(64)     NOT NULL DEFAULT '' COMMENT '当前领取该事件的 Worker 实例 ID',
  locked_until    DATETIME(3)     NULL COMMENT '租约过期时间，防止 Worker 宕机死锁',
  last_error      VARCHAR(512)    NOT NULL DEFAULT '',
  created_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at      DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_event_id (event_id),
  KEY idx_status_retry (status, next_retry_at, id),
  KEY idx_locked_until (status, locked_until),
  KEY idx_tenant_app_created (tenant_id, app_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

---

## 5. 核心事务 SQL 模式：单聊消息持久化与原子 `seq` 分配

禁止使用 `SELECT MAX(seq) + 1 FROM im_message`。标准事务步骤如下（对应任务 11 与任务 12）：

1. 开启 MySQL 事务（`READ COMMITTED` 或默认 `REPEATABLE READ`）。
2. **先查幂等**（或直接依赖唯一索引捕获 `1062 Duplicate entry`）：
   ```sql
   SELECT message_id, seq, created_at
   FROM im_message
   WHERE app_id = ? AND tenant_id = ? AND sender_id = ? AND client_msg_id = ?
   LIMIT 1;
   ```
   若已存在，直接提交/回滚空事务并返回既有 `(message_id, seq, created_at)`。
3. **确保会话行存在**（首次发消息时幂等初始化）：
   ```sql
   INSERT IGNORE INTO im_conversation (tenant_id, app_id, conversation_id, conversation_type, last_seq)
   VALUES (?, ?, ?, 1, 0);
   ```
4. **行锁原子递增 `last_seq`**：
   ```sql
   SELECT last_seq FROM im_conversation
   WHERE app_id = ? AND tenant_id = ? AND conversation_id = ?
   FOR UPDATE;
   -- 计算 nextSeq = last_seq + 1
   UPDATE im_conversation
   SET last_seq = ?, last_message_id = ?, last_message_at = ?
   WHERE app_id = ? AND tenant_id = ? AND conversation_id = ?;
   ```
   *并发死锁预防*：单聊消息只锁 `im_conversation` 单行，不跨多个会话加锁；若并发请求携带同一 `client_msg_id` 在步骤 2 均未命中，先拿到行锁的事务插入成功，后拿到行锁的事务插入 `im_message` 时触发 `uk_app_sender_client_msg` 唯一键冲突，事务回滚后重查并返回原记录（同时不浪费/不污染已提交的 `seq`）。
5. **写入 `im_message`**。
6. **Upsert 双方 `im_conversation_user`**（发送方 `unread_count` 不增加且 `read_seq = GREATEST(read_seq, nextSeq)`；接收方 `unread_count = unread_count + 1` 且 `is_deleted = 0`）。
7. **写入 `im_outbox`**（事件类型 `message.created`，`aggregate_id = conversation_id`）。
8. **提交事务**，随后向发送端返回 `SEND_MESSAGE_ACK`（Server Accepted）。
