# 02 — System architecture

## Design rule

Decouple **Client UI** and **offline outbox** from the **realtime engine**. Migration cost should be “new transport file + protocol mapping,” not “rewrite the app.”

```
┌─────────────────────────────────────────┐
│   Client UI Components (React/Flutter)  │
└────────────────────┬────────────────────┘
                     │ commands / queries (no sockets)
┌────────────────────▼────────────────────┐
│     Offline Outbox Store (SQLite/IDB)   │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│     Abstract Transport Interface        │
│   chatClient.sendMessage()              │
│   chatClient.subscribe(chatId)          │
└──────────┬───────────────────┬──────────┘
           │                   │
 [ Phase 1 ]                   │     [ Phase 2 ]
           ▼                   ▼
┌──────────────────────┐   ┌──────────────────────┐
│ SocketIOTransport.ts │   │ PhoenixTransport.ts  │
└──────────┬───────────┘   └──────────┬───────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┐   ┌──────────────────────┐
│  Node.js + Socket.IO │   │   Elixir + Phoenix   │
│      (Docker)        │   │       Channels       │
└──────────────────────┘   └──────────────────────┘
```

## Client layers (strict)

| Layer | Allowed to know | Forbidden |
| --- | --- | --- |
| UI | Chat list, bubbles, composers, local query models | Socket.IO, Phoenix, Redis, HTTP paths |
| Domain / ChatClient | Outbox, receipts, sync cursor, transport interface | Concrete adapter types except via DI |
| Outbox | SQLite/IndexedDB rows, retry clock | Network frames |
| Transport | Bytes/events over WS | SQL, React state |
| Platform | Login (password / OIDC redirect), token + `deviceId` storage, push | Message business rules |

## Server layers (Phase 1)

```
[ Client Requests (WSS) ]
              │
              ▼
[ Nginx / Cloud Load Balancer ]
  TLS termination, sticky sessions (Socket.IO)
              │
    ┌─────────┴─────────┐
    ▼                   ▼
[ Node Gateway 1 ]   [ Node Gateway 2 ]
  NestJS + Socket.IO
  auth guards
  ChatService (domain providers)
  Redis adapter client
    │                   │
    └─────────┬─────────┘
              │ internal pub/sub
              ▼
     [ Redis ]
       adapter (socket fanout)
       presence / session hints
       idempotency hot cache (optional)
              │
              ▼
     [ PostgreSQL ]
       source of truth: users, chats, messages, receipts
```

Horizontal Socket.IO fanout:

```
[Clients WSS]
      │
      ▼
[Nginx / AWS ALB]
      │
 ┌────┴────┐
 ▼         ▼
[Gateway 1] [Gateway 2]
      │         │
      └──┬──────┘
         │ @socket.io/redis-adapter
         ▼
    [Redis]
```

## Runtime components

| Component | Role | Stateful? |
| --- | --- | --- |
| Web / mobile app | UI + outbox + transport; one local DB **per device** | Yes (local DB) |
| API (HTTP) | Register, login, token refresh, device list/revoke, bootstrap, history | No |
| Gateway (WS) | Live events to **all** of a user’s sockets, presence, typing | Soft (in-memory sockets; shared via Redis) |
| Redis | Pub/sub between gateways, presence TTL, rate-limit counters | Ephemeral |
| PostgreSQL | Canonical messages, membership, idempotency unique index | Durable |
| Object storage (later) | Media blobs | Durable |

Gateways are **disposable**. If a node dies, the client reconnects (sticky or not) and resumes from `lastSeq`.

## Multi-device (Messenger-style)

The account is the identity. Each install (or browser profile) has its own `deviceId` after login. There is no primary device.

- **Login** (`POST /auth/login`) with handle/email + password issues tokens and upserts a `devices` row. A second phone or a browser is a second row, not a takeover unless the user revokes the other session.
- **Fanout:** persist once in Postgres; emit to `user:{userId}` so every connected device of that user sees `message.created` (including the sender’s other devices).
- **Outbox:** only the device that composed the message owns that outbox row. Other devices learn the message from the server (`messageId` / `seq`), not by sharing SQLite.
- **New device:** empty local cache is expected. After auth, HTTP bootstrap + `afterSeq` fills history. Do not require a QR link from a phone.
- **Presence:** user is online if **any** device has a live socket (coarse last-seen).
- **Revoke:** deleting/revoking a device invalidates its refresh tokens; other devices stay signed in.

## Write path (send)

1. UI calls `chatClient.sendText({ chatId, body })`.
2. Client generates `idempotencyKey` (UUIDv7) and `localId`.
3. Outbox inserts row `status=queued` in the same transaction as the local `messages` row `status=pending`.
4. UI renders the bubble immediately.
5. Sync worker calls `transport.sendMessage(envelope)`.
6. Gateway authenticates, rate-limits, then `ChatService.send`.
7. DB: `INSERT ... ON CONFLICT (sender_id, idempotency_key) DO NOTHING RETURNING *` (or upsert returning existing).
8. If insert is new: assign `messageId`, `seq`; publish to Redis channel `chat:{chatId}`.
9. Gateway acks **sender** with `{ localId?, idempotencyKey, messageId, seq, serverTs }`.
10. Outbox marks `acked`; local message `status=sent`.
11. Other gateways emit `message.created` to members in that chat, including the **sender’s other devices**.
12. Recipient clients upsert by `messageId`, ack `message.delivered`.

## Read path (history)

Realtime is not the history API. After connect:

1. HTTP `GET /chats/:id/messages?afterSeq=N` (or `beforeSeq` for scroll-up).
2. WS is for live tail + receipts + typing.
3. Gap detection: if live `seq` jumps, HTTP backfill.

This remains identical in Phoenix (HTTP + Channels).

## Control vs data plane

| Plane | Examples | Transport |
| --- | --- | --- |
| Control | Register, login (credentials), refresh token, device register/revoke | HTTPS |
| Data (sync) | History, chat list bootstrap | HTTPS |
| Data (live) | New messages, receipts, typing, presence | WSS |

Do not send 10k history rows over the socket on connect.

## Environments

| Env | Topology |
| --- | --- |
| Local | `docker compose`: 2 gateways, Redis, Postgres, Nginx sticky |
| Staging | Same + TLS, 2+ nodes |
| Prod Phase 1 | ALB/Nginx, N gateways, Redis (sentinel/cluster when needed), Postgres primary |

Two local gateways are mandatory so Redis adapter bugs show up before production.
