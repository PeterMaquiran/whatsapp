# 01 — Overview

## Goal

Build a WhatsApp-like messenger that:

1. Feels instant on a good network.
2. Still works when the network is bad (queue locally, send later).
3. Does not duplicate messages when the client retries.
4. Can replace Socket.IO with Phoenix Channels without rewriting UI or outbox code.

Phase 1 is **correctness and a clean seam**, not WhatsApp-scale. Scale patterns are designed in, not fully built on day one.

## Product scope (v1)

| In v1 | Explicitly later |
| --- | --- |
| 1:1 chats | Groups / communities |
| Text messages | Media pipeline (upload, transcode, CDN) |
| Delivery + read receipts | End-to-end encryption (Signal protocol) |
| Typing + last-seen (coarse) | Voice/video calls (WebRTC + SFU) |
| Offline send/retry | Multi-device linked sessions like WhatsApp |
| Idempotent send | Message search at scale, status/stories |
| Socket.IO over WSS | Phoenix Channels, Kafka fanout |

## Delivery semantics

We target **at-least-once send from client to server**, made **exactly-once visible** by `idempotency_key`.

| Hop | Guarantee | How |
| --- | --- | --- |
| UI → Outbox | Exactly once in local DB | Insert before paint of “sending” |
| Outbox → Transport | At least once | Retry until ack or terminal fail |
| Transport → Gateway | At least once (WS can drop) | Client retries same key |
| Gateway → DB | Exactly once persist | Unique `(sender_id, idempotency_key)` |
| Gateway → Recipients | At least once | Redis pub/sub; client dedupes by `message_id` |
| Receipts (delivered/read) | At least once | Same idempotent receipt keys |

The user must never see two bubbles for one tap. Server must never insert two rows for one tap. Recipients may receive the same event twice; they key UI on `message_id`.

## WhatsApp mental model (what we copy)

- **Local-first write:** tapping Send writes locally immediately (`client_message_id` / `idempotency_key`).
- **Server assigns canonical `message_id`** (UUID or snowflake). Client maps local id → server id.
- **Ticks:** pending → sent (server ack) → delivered (recipient device ack) → read.
- **Monotonic conversation cursor:** `seq` per chat so clients can gap-fill after reconnect.
- **Sticky realtime session** behind a load balancer; fanout via a shared bus (Redis now, NATS/Kafka later).

What we do **not** copy in v1: sealed sender, multi-device session ratchet, store-and-forward media CDN, spam ML, 2 billion MAU ops.

## Non-goals of the architecture itself

- Binding React/Flutter widgets to `socket.emit`.
- Putting business rules only in the Socket.IO event handlers with no domain layer.
- Using Redis as the source of truth for messages.
- Designing Phoenix-only APIs in Phase 1 (the **interface** is protocol-shaped, not Phoenix-shaped).

## Success criteria for Phase 1

- Kill the process mid-send: after restart, the message is not duplicated.
- Two gateway containers: user A on node 1, user B on node 2, messages still arrive.
- Swap `SocketIOTransport` for a fake `LoopbackTransport` in tests with zero UI changes.
- Every persisted message has `idempotency_key` unique per sender.
