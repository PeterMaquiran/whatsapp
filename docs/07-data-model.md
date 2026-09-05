# 07 — Data model

PostgreSQL is the system of record in Phase 1. Redis is fanout + presence + rate limits. Client SQLite/IDB is a cache + outbox.

## Why Postgres first (B-Tree)

Postgres default indexes are **B-Trees**: excellent for `WHERE chat_id = ? ORDER BY seq` and unique `(sender_id, idempotency_key)`.

**LSM-Trees** (RocksDB, Scylla, Cassandra) win at high ingest and compaction-heavy time series. WhatsApp historically leaned on Erlang + Mnesia/custom; many large chats use Cassandra/Scylla for message logs.

Phase 1: Postgres. Phase 3 (if ingest hurts): messages table → Scylla/Citus/partitioned Postgres. The protocol (`seq`, `idempotency_key`) stays.

## ER (v1)

```
users 1──* devices (many concurrent logins)
users 1──* refresh_tokens
users 1──* chat_members *──1 chats
chats 1──* messages
messages 1──* receipts
chats 1──* chat_members (unread_seq)
```

## SQL

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Account identity is credentials, not a phone number / primary device.
CREATE TABLE users (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  handle         TEXT UNIQUE NOT NULL,
  email          TEXT UNIQUE NOT NULL,
  password_hash  TEXT NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- One row per signed-in install/session. Many devices per user (Messenger-style).
CREATE TABLE devices (
  id            UUID PRIMARY KEY,
  user_id       UUID NOT NULL REFERENCES users(id),
  platform      TEXT NOT NULL, -- web | ios | android | ...
  label         TEXT,          -- "Chrome on Mac", optional
  last_seen_at  TIMESTAMPTZ,
  revoked_at    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX devices_user ON devices (user_id) WHERE revoked_at IS NULL;

CREATE TABLE refresh_tokens (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id),
  device_id     UUID NOT NULL REFERENCES devices(id),
  token_hash    TEXT NOT NULL UNIQUE,
  expires_at    TIMESTAMPTZ NOT NULL,
  revoked_at    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chats (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type          TEXT NOT NULL CHECK (type IN ('direct')),
  current_seq   BIGINT NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chat_members (
  chat_id       UUID NOT NULL REFERENCES chats(id),
  user_id       UUID NOT NULL REFERENCES users(id),
  role          TEXT NOT NULL DEFAULT 'member',
  last_read_seq BIGINT NOT NULL DEFAULT 0,
  joined_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (chat_id, user_id)
);

CREATE INDEX chat_members_user ON chat_members (user_id);

CREATE TABLE messages (
  id                UUID PRIMARY KEY,
  chat_id           UUID NOT NULL REFERENCES chats(id),
  sender_id         UUID NOT NULL REFERENCES users(id),
  idempotency_key   UUID NOT NULL,
  type              TEXT NOT NULL CHECK (type IN ('text')),
  body              TEXT,
  client_ts         TIMESTAMPTZ NOT NULL,
  seq               BIGINT NOT NULL,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (chat_id, seq),
  UNIQUE (sender_id, idempotency_key)
);

CREATE INDEX messages_chat_seq ON messages (chat_id, seq DESC);

CREATE TABLE receipts (
  message_id    UUID NOT NULL REFERENCES messages(id),
  user_id       UUID NOT NULL REFERENCES users(id),
  kind          TEXT NOT NULL CHECK (kind IN ('delivered', 'read')),
  at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (message_id, user_id, kind)
);
```

### Allocating `seq`

```sql
UPDATE chats
SET current_seq = current_seq + 1
WHERE id = $chat_id
RETURNING current_seq;
```

Do this in the **same transaction** as `INSERT INTO messages`. Row lock on `chats` serializes senders in one chat (WhatsApp-like ordering). Fine for 1:1; later groups may need a seq service.

## Redis keys

| Key | Type | TTL | Purpose |
| --- | --- | --- | --- |
| Socket.IO adapter | pub/sub | n/a | Cross-node emit |
| `presence:{user_id}` | set/hash of live `device_id`s | 45s | Online if set non-empty |
| `rl:send:{user_id}` | incr | 1s/1m | Rate limit |
| `typing:{chat_id}:{user_id}` | string | 3s | Typing |

Do not store message bodies in Redis in v1.

## Partitioning (later)

```sql
-- when a chat is huge, seq index still works
-- when the table is huge:
-- PARTITION BY RANGE (created_at)  -- weak for chat-seq reads
-- better: hash by chat_id (Citus) or Scylla PK (chat_id, seq)
```

Analyze `messages_chat_seq` vs table size before migrating to LSM.

## Direct chat uniqueness

```sql
-- canonical pair
CREATE UNIQUE INDEX direct_chat_pair ON chats (id) WHERE type = 'direct'; -- insufficient

CREATE TABLE direct_pairs (
  user_lo UUID NOT NULL,
  user_hi UUID NOT NULL,
  chat_id UUID NOT NULL REFERENCES chats(id),
  PRIMARY KEY (user_lo, user_hi),
  CHECK (user_lo < user_hi)
);
```

Creates one 1:1 thread per pair.
