# 01 — Overview

## Goal

Build a messenger that **feels** like WhatsApp (instant send, offline queue, receipts) but **authenticates** like Facebook Messenger: an account with credentials, usable on several devices at once.

1. Feels instant on a good network.
2. Still works when the network is bad (queue locally, send later).
3. Does not duplicate messages when the client retries.
4. Can replace Socket.IO with Phoenix Channels without rewriting UI or outbox code.
5. Lets the same user sign in on phone, web, and tablet with the same credentials — no primary-phone lock-in.

Phase 1 is **correctness and a clean seam**, not WhatsApp-scale. Scale patterns are designed in, not fully built on day one.

## Identity and devices (not WhatsApp)

WhatsApp is phone-number identity plus a **primary device** (and later QR-linked companions with a special session ratchet). **This product is not that.**

| | This product (v1) | WhatsApp | Messenger-like target |
| --- | --- | --- | --- |
| Identity | Account: handle/email + password and/or Google (Keycloak later, same seam) | Phone number | Email / phone / password |
| Sign-in | Credentials or Google on any device | Primary phone; companions linked | Same account, many logins |
| Concurrent sessions | Yes — web + mobile at once | Constrained / linked | Yes |
| History on a new device | Server is source of truth; HTTP sync after login | Tied to device / backup / linking | Cloud history after login |
| Local outbox | Per device (each has its own SQLite/IDB) | Per device | Per device |

v1 **includes**: register, login (password and Google), refresh, logout, register a `deviceId` per install/session, revoke a device, fanout live events to **all** of the user’s connected sockets.

v1 **does not include**: WhatsApp-style QR companion linking, a privileged “primary” phone, or Signal multi-device session keys. Those stay out because we are not copying WhatsApp’s identity model.

A new laptop is a new `devices` row after a successful login (password or Google), then history comes from Postgres over HTTP. The user is not “this phone.” IdP tokens never ride the socket; see ADR-012.

## Product scope (v1)

| In v1 | Explicitly later |
| --- | --- |
| Credential auth (register / login / refresh / logout) + Google OAuth | Keycloak / Apple / other OIDC (same `identities` + `issueSession`), phone OTP |
| Same account on multiple devices at once | WhatsApp-style QR-linked companion sessions |
| 1:1 chats | Groups / communities |
| Text messages | Media pipeline (presigned PUT; **TUS** for large / unstable; transcode; CDN) |
| Delivery + read receipts | End-to-end encryption (Signal protocol) |
| Typing + last-seen (coarse) | Voice/video calls (WebRTC + SFU) |
| Offline send/retry (per device) | Message search at scale, status/stories |
| Idempotent send + Socket.IO over WSS | Phoenix Channels, Kafka fanout |

## Delivery semantics

We target **at-least-once send from client to server**, made **exactly-once visible** by `idempotencyKey`.

| Hop | Guarantee | How |
| --- | --- | --- |
| UI → Outbox | Exactly once in local DB | Insert before paint of “sending” |
| Outbox → Transport | At least once | Retry until ack or terminal fail |
| Transport → Gateway | At least once (WS can drop) | Client retries same key |
| Gateway → DB | Exactly once persist | Unique `(sender_id, idempotency_key)` |
| Gateway → Recipients | At least once | Redis pub/sub; client dedupes by `messageId` |
| Receipts (delivered/read) | At least once | Same idempotent receipt keys |

The user must never see two bubbles for one tap. Server must never insert two rows for one tap. Recipients may receive the same event twice; they key UI on `messageId`.

## Chat mental model (what we copy from WhatsApp-like UX)

Realtime *behavior*, not WhatsApp *accounts*:

- **Local-first write:** tapping Send writes locally immediately (`localId` / `idempotencyKey`).
- **Server assigns canonical `messageId`** (UUID or snowflake). Client maps local id → server id.
- **Ticks:** pending → sent (server ack) → delivered (recipient **user** has the message on at least one device) → read (recipient **user** opened it). Receipts are per `userId`, not per device.
- **Monotonic conversation cursor:** `seq` per chat so clients can gap-fill after reconnect.
- **Sticky realtime session** behind a load balancer; fanout via a shared bus (Redis now, NATS/Kafka later).
- **Fanout to every logged-in device** of a member (`user:{userId}` rooms), like Messenger — not “only the primary phone.”

What we do **not** copy: phone-as-identity, primary-device lock, sealed sender, multi-device session ratchet, store-and-forward media CDN, spam ML, 2 billion MAU ops.

## Non-goals of the architecture itself

- Binding React/Flutter widgets to `socket.emit`.
- Putting business rules only in the Socket.IO event handlers with no domain layer.
- Using Redis as the source of truth for messages.
- Designing Phoenix-only APIs in Phase 1 (the **interface** is protocol-shaped, not Phoenix-shaped).

## Success criteria for Phase 1

- Kill the process mid-send: after restart, the message is not duplicated.
- Two gateway containers: user A on node 1, user B on node 2, messages still arrive.
- Same user logged in on two clients: a message sent from one appears on the other without a second tap.
- New device after credential login hydrates chat list + history from HTTP (empty local DB is fine).
- Swap `SocketIOTransport` for a fake `LoopbackTransport` in tests with zero UI changes.
- Every persisted message has `idempotencyKey` unique per sender.
