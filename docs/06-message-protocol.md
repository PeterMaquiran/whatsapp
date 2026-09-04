# 06 — Message protocol

Protocol is **transport-agnostic**. Socket.IO event names ≈ Phoenix event names. Payloads are JSON in Phase 1; Protobuf is a later encoding, not a different model.

`protocol_version: 1`

## Envelope

Every WS frame (client → server and server → client):

```json
{
  "v": 1,
  "event": "message.send",
  "request_id": "uuid",
  "ts": 1725460000000,
  "payload": {}
}
```

`request_id` is for tracing (OpenTelemetry), not idempotency. Do not reuse `request_id` as `idempotency_key`.

## Client → server

### `message.send`

```json
{
  "chat_id": "uuid",
  "idempotency_key": "uuid",
  "local_id": "uuid",
  "type": "text",
  "body": "hello",
  "client_ts": 1725460000000,
  "reply_to_message_id": null
}
```

Ack:

```json
{
  "ok": true,
  "duplicate": false,
  "idempotency_key": "uuid",
  "local_id": "uuid",
  "message_id": "uuid",
  "chat_id": "uuid",
  "seq": 1042,
  "server_ts": 1725460000123
}
```

Error ack:

```json
{
  "ok": false,
  "code": "FORBIDDEN",
  "message": "not a member",
  "idempotency_key": "uuid"
}
```

### `receipt.delivered` / `receipt.read`

```json
{
  "chat_id": "uuid",
  "message_id": "uuid",
  "idempotency_key": "uuid"
}
```

`read` may be compacted: `{ "chat_id", "up_to_seq", "idempotency_key" }` meaning all messages `seq <= up_to_seq` from others are read.

### `typing.set`

```json
{ "chat_id": "uuid", "is_typing": true }
```

Ephemeral. No DB. No idempotency. Redis pub/sub only. TTL ~3s.

### `presence.ping` (optional)

Heartbeat if you do not trust socket disconnect. Presence is still Redis TTL.

## Server → client

### `message.created`

Same shape as persisted message:

```json
{
  "message_id": "uuid",
  "chat_id": "uuid",
  "sender_id": "uuid",
  "idempotency_key": "uuid",
  "type": "text",
  "body": "hello",
  "client_ts": 1725460000000,
  "server_ts": 1725460000123,
  "seq": 1042,
  "reply_to_message_id": null
}
```

Sender receives this **or** only the ack; Phase 1: **ack + echo**. Client must merge by key (see outbox).

### `message.delivered` / `message.read`

```json
{
  "chat_id": "uuid",
  "message_id": "uuid",
  "user_id": "uuid",
  "at": 1725460000500,
  "up_to_seq": 1042
}
```

### `typing`

```json
{ "chat_id": "uuid", "user_id": "uuid", "is_typing": true }
```

### `presence`

```json
{ "user_id": "uuid", "state": "online" | "offline", "last_seen_at": 1725460000000 }
```

Coarse last-seen only in v1 (privacy settings later).

## HTTP (sync)

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/auth/login` | returns access + refresh |
| `GET` | `/chats` | membership + last message preview + unread |
| `GET` | `/chats/:id/messages?after_seq=&limit=` | forward sync |
| `GET` | `/chats/:id/messages?before_seq=&limit=` | history up |
| `POST` | `/chats` | create 1:1 |
| `POST` | `/messages` | optional fallback; same idempotency |

Pagination is **seq-based**, not offset-based.

## Seq rules

- `seq` is `INTEGER` per `chat_id`, strictly increasing by 1 for each **visible** message.
- Never reuse seq. If a transaction fails after bump, skip (gap) rather than duplicate. Clients treat gaps as “fetch HTTP.”
- Receipts do not consume seq.
- Deletes/edits (later) are new events with their own seq **or** a separate `chat_events` stream. Prefer `chat_events(seq)` from day one if you want edits without schema pain.

Recommended future-proof table: `chat_events` where `type=message_created|message_edited|message_deleted` and `payload` JSON. v1 only inserts `message_created`.

## Ordering

- UI order: `seq` when present, else `client_ts` for pending.
- Do not order purely by `server_ts` (clocks). Seq is king.

## Size limits (v1)

| Field | Limit |
| --- | --- |
| `body` | 4096 UTF-8 bytes |
| frames | 64 KiB |
| history page | 50–100 |

## Protobuf (later)

Same fields, `proto3`. Transport capability `binary: true`. Do not mix JSON and proto on one connection without versioning.
