# 06 — Message protocol

Protocol is **transport-agnostic**. Socket.IO event names ≈ Phoenix event names. Payloads are JSON in Phase 1; Protobuf is a later encoding, not a different model.

`protocolVersion: 1`

**Naming (ADR-007):** JSON keys and TypeScript fields are **camelCase**. Postgres/SQLite columns are **snake_case** (`chat_id` ↔ `chatId`). Event names stay dotted (`message.send`).

## Envelope

Every WS frame (client → server and server → client):

```json
{
  "v": 1,
  "event": "message.send",
  "requestId": "uuid",
  "ts": 1725460000000,
  "payload": {}
}
```

`requestId` is for tracing (OpenTelemetry), not idempotency. Do not reuse `requestId` as `idempotencyKey`.

## Client → server

### `message.send`

```json
{
  "chatId": "uuid",
  "idempotencyKey": "uuid",
  "localId": "uuid",
  "type": "text",
  "body": "hello",
  "clientTs": 1725460000000,
  "replyToMessageId": null
}
```

Ack:

```json
{
  "ok": true,
  "duplicate": false,
  "idempotencyKey": "uuid",
  "localId": "uuid",
  "messageId": "uuid",
  "chatId": "uuid",
  "seq": 1042,
  "serverTs": 1725460000123
}
```

Error ack:

```json
{
  "ok": false,
  "code": "FORBIDDEN",
  "message": "not a member",
  "idempotencyKey": "uuid"
}
```

### `receipt.delivered` / `receipt.read`

```json
{
  "chatId": "uuid",
  "messageId": "uuid",
  "idempotencyKey": "uuid"
}
```

`read` may be compacted: `{ "chatId", "upToSeq", "idempotencyKey" }` meaning all messages `seq <= upToSeq` from others are read.

### `typing.set`

```json
{ "chatId": "uuid", "isTyping": true }
```

Ephemeral. No DB. No idempotency. Redis pub/sub only. TTL ~3s.

### `presence.ping` (optional)

Heartbeat if you do not trust socket disconnect. Presence is still Redis TTL.

## Server → client

### `message.created`

Same shape as persisted message:

```json
{
  "messageId": "uuid",
  "chatId": "uuid",
  "senderId": "uuid",
  "idempotencyKey": "uuid",
  "type": "text",
  "body": "hello",
  "clientTs": 1725460000000,
  "serverTs": 1725460000123,
  "seq": 1042,
  "replyToMessageId": null
}
```

Sender receives this **or** only the ack; Phase 1: **ack + echo**. Client must merge by key (see outbox).

### `message.delivered` / `message.read`

```json
{
  "chatId": "uuid",
  "messageId": "uuid",
  "userId": "uuid",
  "at": 1725460000500,
  "upToSeq": 1042
}
```

### `typing`

```json
{ "chatId": "uuid", "userId": "uuid", "isTyping": true }
```

### `presence`

```json
{ "userId": "uuid", "state": "online" | "offline", "lastSeenAt": 1725460000000 }
```

Coarse last-seen only in v1 (privacy settings later).

## HTTP (sync)

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/auth/register` | handle, email, password → user |
| `POST` | `/auth/login` | credentials + `deviceId` / `platform` → access + refresh; upserts device |
| `GET` | `/auth/oidc/:provider` | OIDC start (Nest). v1 `provider=google`. Later `keycloak`. Query: `deviceId` / `platform` |
| `GET` | `/auth/oidc/:provider/callback` | IdP redirect; upsert `users` + `identities`; set refresh cookie; issue **our** access JWT |
| `POST` | `/auth/refresh` | rotated refresh, same `deviceId` |
| `POST` | `/auth/logout` | revoke this device’s refresh (optional `all: true`) |
| `GET` | `/devices` | list sessions for the account |
| `DELETE` | `/devices/:id` | revoke another (or this) device |
| `GET` | `/chats` | membership + last message preview + unread |
| `GET` | `/chats/:id/messages?afterSeq=&limit=` | forward sync |
| `GET` | `/chats/:id/messages?beforeSeq=&limit=` | history up |
| `POST` | `/chats` | create 1:1 |
| `POST` | `/messages` | optional fallback; same idempotency |

Pagination is **seq-based**, not offset-based.

## Seq rules

- `seq` is `INTEGER` per chat (`chats` row / `chatId`), strictly increasing by 1 for each **visible** message.
- Never reuse seq. If a transaction fails after bump, skip (gap) rather than duplicate. Clients treat gaps as “fetch HTTP.”
- Receipts do not consume seq.
- Deletes/edits (later) are new events with their own seq **or** a separate `chat_events` stream. Prefer `chat_events(seq)` from day one if you want edits without schema pain.

Recommended future-proof table: `chat_events` where `type=message_created|message_edited|message_deleted` and `payload` JSON. v1 only inserts `message_created`.

## Ordering

- UI order: `seq` when present, else `clientTs` for pending.
- Do not order purely by `serverTs` (clocks). Seq is king.

## Size limits (v1)

| Field | Limit |
| --- | --- |
| `body` | 4096 UTF-8 bytes |
| frames | 64 KiB |
| history page | 50–100 |

## Protobuf (later)

Same fields, `proto3` (json_name camelCase). Transport capability `binary: true`. Do not mix JSON and proto on one connection without versioning.
