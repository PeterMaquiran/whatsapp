# 05 — Idempotency

## Problem

WebSockets retry. Users tap twice. The OS kills the app after send but before ack. Without a stable key, you get duplicate bubbles and duplicate DB rows.

## Rule

**The client generates `idempotency_key` once, at outbox insert. Every retry, reconnect, and echo uses that same key. The server treats `(sender_id, idempotency_key)` as unique.**

Never hash the message body as the key (edits, identical “ok” texts, retries after edit would collide or miss).

## Key format

- UUIDv7 (time-ordered) or ULID.
- 128-bit, generated locally.
- Optional prefix `msg_` for logs only; store raw UUID in DB.

Not a snowflake from the server — the client must have the key **before** the network exists.

## Envelope fields

```json
{
  "idempotency_key": "018f3c2a-9c1e-7b00-8000-0123456789ab",
  "local_id": "dev-local-uuid",
  "chat_id": "...",
  "type": "text",
  "body": "hello",
  "client_ts": 1725460000000
}
```

- `idempotency_key`: **dedup identity** (global per sender).
- `local_id`: UI/outbox row id (may equal the key; keeping both lets UI change storage without changing protocol).
- `message_id`: **server** canonical id, assigned on first persist.
- `seq`: per-chat monotonic integer assigned on first persist.

## Server persist (Postgres)

```sql
CREATE UNIQUE INDEX messages_sender_idempotency
  ON messages (sender_id, idempotency_key);

-- first write
INSERT INTO messages (
  id, chat_id, sender_id, idempotency_key, type, body, client_ts, seq, created_at
) VALUES (
  $id, $chat_id, $sender_id, $key, $type, $body, $client_ts, $seq, now()
)
ON CONFLICT (sender_id, idempotency_key)
DO NOTHING
RETURNING *;
```

If `RETURNING` is empty, `SELECT` by `(sender_id, idempotency_key)` and return the **original** row (`duplicate: true` in ack).

**Do not update body on conflict.** First write wins. That keeps retries safe.

## Race: two in-flight identical keys

Two connections, same user, same key (multi-tab):

1. Both INSERT.
2. One wins unique index; one hits conflict.
3. Loser reads winner row and acks the same `message_id`.
4. Fanout happens **once** (only the winner publishes). Loser must **not** publish again.

Implement publish inside the same unit of work:

```
BEGIN
  INSERT ... ON CONFLICT DO NOTHING RETURNING
  if inserted:
    bump chat.seq
    commit
    publish
  else:
    rollback-to-select
    ack duplicate, no publish
END
```

Use `INSERT ... ON CONFLICT DO NOTHING RETURNING` + `xmax`/`xmin` tricks, or a separate `message_idempotency` table with `INSERT` then insert message only if new.

Recommended explicit table:

```sql
CREATE TABLE message_idempotency (
  sender_id        UUID NOT NULL,
  idempotency_key  UUID NOT NULL,
  message_id       UUID NOT NULL REFERENCES messages(id),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (sender_id, idempotency_key)
);
```

Flow:

1. Try insert idempotency row with a **preallocated** `message_id`.
2. If conflict, return existing `message_id`.
3. If ok, insert `messages` with that id and next `seq`.

Preallocation avoids a gap if step 3 fails: handle with transactional outbox on the **server** (see below) or retry insert message with same id.

## Server-side outbox (Phase 1 light, Phase 2 strong)

Gateway publish after commit can still lose the Redis message. Recipients then depend on HTTP gap fill (`seq`). That is acceptable for v1 if `seq` is strictly monotonic.

Later: `server_outbox` table + publisher worker (Kafka/NATS). Same idea as the client outbox.

## Redis cache (optional)

`SET idem:{sender_id}:{key} {message_id} NX EX 86400` **is not enough**. Redis is not the source of truth. Unique index in Postgres is mandatory. Redis may be a fast path after the row exists.

## Replay window

Keep idempotency rows for the life of the message (no TTL). Disk is cheap vs user-visible dupes. If you ever TTL, it must exceed max client retry (weeks, not minutes).

## Receipts

```sql
CREATE UNIQUE INDEX receipts_unique
  ON receipts (message_id, user_id, kind);
-- kind: delivered | read
```

Client `idempotency_key` on receipts is optional if this unique pair exists; still send a key so retries of the RPC are cheap.

## HTTP vs WS

If you add `POST /messages` as a fallback:

```
Idempotency-Key: <uuid>
```

Same server function as WS `message.send`. One domain method: `ChatService.send(cmd)`.

## Client display dedup

Index local rows by:

1. `idempotency_key` (own sends)
2. `message_id` (everyone)

Merge: echo of own message updates the pending row; never inserts a second.

## Test matrix

| Case | Expected |
| --- | --- |
| Double tap, two keys | Two messages (correct) |
| Double tap, same key (bug) | One message |
| Retry after timeout | One row, `duplicate: true` on second ack |
| Two tabs same key | One row, one fanout |
| Recipient gets event twice | One bubble |
| Reinstall app, new keys | New messages (acceptable; old pending lost with local DB) |
