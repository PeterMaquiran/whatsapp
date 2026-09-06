# 04 — Offline outbox

## Role

The outbox is the **local source of truth for unconfirmed writes on this device**. The network is a delivery mechanism, not the store the composer writes to. Another laptop logged into the same account has its **own** outbox; it never shares this SQLite/IDB. Server Postgres is how the other device sees the message.

Same pattern on:

- Mobile: SQLite (SQLCipher later if needed)
- Web: IndexedDB (Dexie or SQLite WASM — pick one and wrap)

## Local tables (SQLite)

```sql
CREATE TABLE local_messages (
  local_id            TEXT PRIMARY KEY,
  chat_id             TEXT NOT NULL,
  message_id          TEXT,              -- null until server ack
  idempotency_key     TEXT NOT NULL UNIQUE,
  sender_id           TEXT NOT NULL,
  type                TEXT NOT NULL,
  body                TEXT,
  client_ts           INTEGER NOT NULL,
  server_ts           INTEGER,
  seq                 INTEGER,
  status              TEXT NOT NULL,     -- pending | sent | delivered | read | failed_terminal
  created_at          INTEGER NOT NULL
);

CREATE TABLE outbox (
  id                  TEXT PRIMARY KEY,
  kind                TEXT NOT NULL,     -- message.send | receipt.delivered | receipt.read
  idempotency_key     TEXT NOT NULL UNIQUE,
  payload_json        TEXT NOT NULL,
  status              TEXT NOT NULL,     -- queued | in_flight | acked | failed_terminal
  attempt_count       INTEGER NOT NULL DEFAULT 0,
  next_attempt_at     INTEGER NOT NULL,
  last_error          TEXT,
  created_at          INTEGER NOT NULL,
  updated_at          INTEGER NOT NULL
);

CREATE INDEX outbox_ready ON outbox (status, next_attempt_at);

CREATE TABLE sync_cursors (
  chat_id             TEXT PRIMARY KEY,
  last_seq            INTEGER NOT NULL DEFAULT 0
);
```

`local_messages.idempotency_key` = `outbox.idempotency_key` for sends (SQL). Application objects use `idempotencyKey`.

## Send pipeline

```
tap Send
  → tx {
      insert local_messages (pending)
      insert outbox (queued)
    }
  → UI observes local_messages
  → OutboxWorker
       if connected and next_attempt_at <= now
         status = in_flight
         transport.sendMessage(...)
         on ack: outbox acked, local messageId+seq, status=sent
         on retryable: queued, attempt++, exponential backoff
         on terminal: failed_terminal
```

**Single worker** per device (mutex). Parallel sends to **different** chats are allowed; same chat should preserve client order by `clientTs` / queue order.

## Backoff

```
attempts 1..n: min(2^attempt * 500ms, 30s) + jitter
```

Cap attempts only for terminal validation errors, not for network.

## In-flight crash

If the app dies while `in_flight`:

- On boot, treat `in_flight` older than `serverAckTimeoutMs * 2` as `queued`.
- Retry **same** `idempotencyKey`.
- Server unique constraint prevents a second row.

## Mapping server id

When ack arrives:

```
UPDATE local_messages
SET message_id = ?, seq = ?, server_ts = ?, status = 'sent'
WHERE idempotency_key = ?
```

Incoming `message.created` for **own** messages: upsert on `idempotencyKey` first, then `messageId`. Avoids a duplicate bubble (optimistic + echo).

## Receipts also use the outbox

Delivered/read must survive offline:

- `kind = receipt.read`, key = `read:{chatId}:{userId}:{upToSeq}` or UUID plus unique server constraint.
- Do not fire-and-forget receipts on WS.

## What the UI queries

UI **never** reads `outbox`. It reads `local_messages` joined with incoming replicas. A sync projector applies WS/HTTP into `local_messages`.

## Web vs mobile

| | SQLite | IndexedDB |
| --- | --- | --- |
| Transactions | Strong | OK if you wrap carefully |
| Crash safety | Good | Good enough for v1 |
| Shared worker | Native | Dedicated Worker recommended so tabs share one outbox |

Multiple browser tabs: one leader tab/worker owns the outbox (Web Locks API).
