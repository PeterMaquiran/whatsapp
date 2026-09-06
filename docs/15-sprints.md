# 15 — Sprints and tasks (Phase 1)

Engineering build order for GitHub: **milestones = sprints**, **issues = tasks**. Copy titles as-is. Suggested labels: `web` · `gateway` · `shared` · `infra`.

This expands the one-page list in [13 — Learning roadmap](./13-learning-roadmap.md). Do not start the backlog until Sprint 10 is boringly true. Phase 1 success criteria live in [01 — Overview](./01-overview.md).

## Repo shape

One repository for now: **Next.js web + NestJS gateway**. Split later if needed.

| Path | Role |
| --- | --- |
| `apps/web` | Next.js App Router. UI, credential session storage, IndexedDB outbox. |
| `apps/gateway` | Long-running **NestJS** HTTP + Socket.IO. Domain in providers (`ChatService`). Not serverless route handlers. |
| `packages/protocol` | Shared JSON protocol types (doc 06). |
| `packages/chat-client` | `ChatTransport`, `ChatClient`, outbox worker. UI never imports Socket.IO. |

Socket.IO + two gateways + Redis adapter needs a process that stays up. Next.js API routes are for later convenience only; they are not the live plane.

**Invariant:** components talk to `ChatClient`, never `socket.emit` or Phoenix.

---

## Sprint 1 — Repo & local platform

**Goal:** Monorepo runs; health check is green.

| # | Task | Done when |
| --- | --- | --- |
| 1 | Scaffold monorepo (Next.js + NestJS gateway + workspaces) | `web` and `gateway` start; shared package imports work |
| 2 | Docker Compose: Postgres + Redis | Both healthy; gateway reads `DATABASE_URL` / `REDIS_URL` |
| 3 | Gateway `/healthz` and `/readyz` | Liveness always; readiness checks Postgres (+ Redis when used); Compose `readyz` is 200 |
| 4 | CI: lint + typecheck | PR CI runs on `web`, `gateway`, `packages` |

---

## Sprint 2 — Credential auth & devices

**Goal:** Messenger-style login, many devices, no phone/QR. See [11 — Security](./11-security.md), [07 — Data model](./07-data-model.md).

| # | Task | Done when |
| --- | --- | --- |
| 5 | Postgres: `users`, `devices`, `refresh_tokens`, `identities` | Migrate up/down on a clean DB; `password_hash` nullable; `identities(provider, subject)` unique |
| 6 | `POST /auth/register` and `POST /auth/login` | Handle/email + password (Argon2id); login upserts `devices`; JWT claims `sub` + `device_id`; hashed refresh **scoped to device**; two logins = two device rows. Session minting isolated in `AuthService.issueSession` |
| 7 | `POST /auth/refresh` and `POST /auth/logout` | Logout this device vs `all: true`; revoked refresh rejected; other devices stay signed in |
| 8 | `GET /devices` and `DELETE /devices/:id` | Revoke one session; others still work |
| 9 | Next.js: register / login / session | Tokens + `device_id` only in the platform layer; sign up, sign in, refresh, sign out in the browser |
| 9a | OIDC on Nest (`/auth/oidc/:provider`) + Google config | After password JWT works. Provider interface + Google. Upsert `identities`; `issueSession`; socket still uses our JWT. Same email **links**, does not duplicate |
| 9b | Next.js: Continue with Google | Redirect to Nest `/auth/oidc/google`; return with session; OIDC-only user can pick/edit `handle` if generated |

---

## Sprint 3 — Chats + idempotent send (HTTP first)

**Goal:** Canonical messages in Postgres. No Socket.IO yet. See [05 — Idempotency](./05-idempotency.md).

| # | Task | Done when |
| --- | --- | --- |
| 10 | Schema: `chats`, `chat_members`, `direct_pairs`, `messages` | Unique `(sender_id, idempotency_key)` and `(chat_id, seq)`; one 1:1 pair cannot create two threads |
| 11 | `POST /chats` (1:1) and `GET /chats` | Membership + last preview + unread; A creates chat with B; both list it |
| 12 | `ChatService.send` + `POST /messages` | Same txn: bump `chats.current_seq`, insert; conflict → original row, `duplicate: true`; never update body on conflict; sender from token; body ≤ 4096 bytes; membership check; double POST same key → one row **without WS** |
| 13 | `GET /chats/:id/messages?after_seq=&before_seq=` | Seq pagination, not offset; empty after last seq |
| 14 | Next.js: chat list + thread (HTTP only) | Two browsers (two users) send via HTTP and see history after refresh |

---

## Sprint 4 — Transport seam + local outbox

**Goal:** UI never knows Socket.IO. Local-first send. See [03 — Transport](./03-transport-abstraction.md), [04 — Outbox](./04-offline-outbox.md).

| # | Task | Done when |
| --- | --- | --- |
| 15 | `packages/chat-client`: `ChatTransport` + `LoopbackTransport` | Contract tests: ack, duplicate key, retry same key; no Redis |
| 16 | IndexedDB outbox + `local_messages` + `sync_cursors` | Dexie or SQLite WASM (pick one); send inserts outbox + pending message in one tx; UI reads `local_messages` only |
| 17 | Outbox worker + `HttpTransport` | UUIDv7 `idempotency_key` at insert; never regenerate; backoff; `in_flight` crash → requeue same key; kill mid-send, restart, still one server row |
| 18 | Wire composer → `chatClient.sendText` | Bubble paints immediately (`pending`); offline queue then send when online; no duplicate bubbles |

---

## Sprint 5 — Socket.IO live path

**Goal:** Live ack + echo; domain stays in `ChatService`. See [08 — Phase 1 Socket.IO](./08-phase-1-socketio.md), [06 — Protocol](./06-message-protocol.md).

| # | Task | Done when |
| --- | --- | --- |
| 19 | Socket.IO on Nest gateway; auth in handshake guard / `io.use` | JWT handshake; rooms `user:{userId}`, `chat:{chatId}`, `device:{deviceId}`; join chat only after membership; bad token rejected; persist stays in `ChatService` |
| 20 | `message.send` → `ChatService` → ack (+ echo `message.created`) | Thin `ChatGateway`; persist then emit; envelope `v` + `request_id` (tracing, not idempotency); ack has `message_id` + `seq`; HTTP duplicate still one row |
| 21 | `SocketIOTransport` implementing `ChatTransport` | Ack timeout; `TIMEOUT` ≠ terminal fail; swap loopback → Socket.IO with **zero UI changes** |
| 22 | HTTP bootstrap after connect | Chat list + `after_seq`; no history dump on the socket; reconnect fills gaps via HTTP, then live tail |

---

## Sprint 6 — Multi-device + two gateways

**Goal:** Phase 1 fanout and hydration actually work.

| # | Task | Done when |
| --- | --- | --- |
| 23 | Redis adapter + Compose scale gateway × 2 | User A on node 1, user B on node 2, messages arrive |
| 24 | Fanout to all devices of a user | Persist once; emit to `user:{userId}`; outbox stays per device; same account, two browsers: send on one appears on the other |
| 25 | New device hydrates from HTTP | Empty IDB after login is OK; third login sees chat list + history |
| 26 | Seq gap detection + HTTP backfill | Live `seq` jump → `chat.seq.gap` → `after_seq`; kill a node, client repairs without duplicates |

---

## Sprint 7 — Receipts, typing, presence

**Goal:** WhatsApp-like ticks; receipts per `user_id`, not per device.

| # | Task | Done when |
| --- | --- | --- |
| 27 | Delivered / read receipts (DB + outbox) | Unique `(message_id, user_id, kind)`; compact read `up_to_seq` OK; ticks pending → sent → delivered → read; survive offline |
| 28 | Typing (Redis TTL, no DB) | Shows in open 1:1; expires ~3s |
| 29 | Presence: online if any device; coarse last-seen | Redis set of `device_id`s; one device disconnects, user stays online if another is connected |
| 30 | Chat UI: ticks, typing, last-seen | Still no sockets in components; 1:1 text feels like a messenger |

---

## Sprint 8 — Edge, abuse, resilience

**Goal:** Sticky LB, caps, breaker. See [10 — Resilience](./10-resilience-and-scale.md).

| # | Task | Done when |
| --- | --- | --- |
| 31 | Nginx sticky (ip_hash or cookie) in front of 2 gateways | Socket.IO handshake works through Nginx |
| 32 | Rate limits (Redis): send, login, typing | e.g. send 20/10s → `RATE_LIMITED`; outbox backoff, same key; persist still correct |
| 33 | Postgres circuit breaker + load shed sockets | Dead PG → `UNAVAILABLE`; no insert storm |
| 34 | Authz + size limits | Membership on every send/history; no client `sender_id`; 64 KiB frames; non-member is `FORBIDDEN` |

---

## Sprint 9 — Observability

**Goal:** One `message.send` through traces, logs, metrics. Never put message `body` in signals. See [12 — Observability](./12-observability.md).

| # | Task | Done when |
| --- | --- | --- |
| 35 | OTel SDK in gateway (OTLP only) | `request_id` = `trace_id` on send; span names like `chat.send`; app talks to Collector only |
| 36 | Compose: Collector + Tempo + Loki + Prometheus + Grafana | Grafana Explore: log → Tempo for one send; RED metrics exist |

---

## Sprint 10 — Phase 1 freeze

**Goal:** Doc 01 success criteria are true.

| # | Task | Done when |
| --- | --- | --- |
| 37 | Web Locks: one outbox worker across tabs | Two tabs, one account, no double send |
| 38 | Token refresh: disconnect/reconnect with new access token | 15 min expiry does not lose the outbox |
| 39 | Acceptance tests (doc 01) | Kill mid-send; two gateways; two devices; new device HTTP hydrate; Loopback in tests; unique idempotency — automated or a runbook you actually ran |

---

## Backlog (not v1)

Do not start until Sprint 10 is solid.

| # | Task | Notes |
| --- | --- | --- |
| 40 | Phoenix `ChatTransport` behind a flag | Same protocol; see [09](./09-phase-2-phoenix.md) |
| 40a | Keycloak OIDC (`provider=keycloak`) | Same `/auth/oidc/:provider` + `identities`. Env issuer/client. Do not put Keycloak JWTs on the socket. See ADR-012 |
| 41 | Media: presigned PUT; TUS for large/flaky | `message.send` only references a `ready` `media_id` |
| 42 | Groups | Fanout changes; seq still per chat |
| 43 | Kafka/NATS server outbox | Client protocol unchanged |
| 44 | E2E (Signal/MLS) | Do not market TLS as E2E |
| 45 | Calls (WebRTC + SFU) | UDP/QUIC; out of Phase 1 |

---

## GitHub

1. Create milestones named exactly as the sprint headings (`Sprint 1 — Repo & local platform` … `Sprint 10 — Phase 1 freeze`, plus `Backlog`).
2. Create one issue per numbered row; set the milestone; add labels.
3. Optional Project board: To do / In progress / Done, filtered by milestone.

Work sprints in order. Inside a sprint, finish gateway/schema tasks before the matching Next.js UI when both exist.
