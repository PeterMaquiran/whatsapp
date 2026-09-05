# 14 — Decisions (ADR)

## ADR-001: Socket.IO first, Phoenix later

**Status:** accepted

**Context:** Need realtime quickly; team/JS ecosystem; migration planned.

**Decision:** Phase 1 gateways in Node + Socket.IO + Redis adapter. Abstract `ChatTransport`. Phase 2 Phoenix Channels, same protocol and Postgres.

**Consequences:** Sticky LB; adapter semantics; discipline on isolation. Phoenix is not a rewrite.

## ADR-002: PostgreSQL as message source of truth

**Status:** accepted

**Context:** LSM stores scale ingest; they add ops cost.

**Decision:** Postgres + B-Tree `(chat_id, seq)` until measured pain. Redis not allowed to allocate `seq` or store canonical bodies.

## ADR-003: Client-generated idempotency_key

**Status:** accepted

**Context:** At-least-once WS/HTTP.

**Decision:** UUIDv7 at outbox insert; unique `(sender_id, idempotency_key)`; first write wins; `duplicate` flag on ack.

## ADR-004: Seq per chat in Postgres row lock

**Status:** accepted

**Context:** Need gap-detectable ordering.

**Decision:** `chats.current_seq` updated in the send transaction. Recipients repair via HTTP.

## ADR-005: No E2E in v1

**Status:** accepted

**Context:** Signal/MLS dominates schedule.

**Decision:** TLS + authz only. Do not market as E2E. Keep `body` replaceable by ciphertext later.

## ADR-006: History over HTTP, live over WS

**Status:** accepted

**Decision:** Sockets are not a query API. Prevents painful Phoenix and mobile battery issues.

## ADR-007: JSON protocol v1

**Status:** accepted

**Decision:** Field-stable JSON; Protobuf later as encoding. Event names stay stable across Socket.IO and Phoenix.

## ADR-008: Credential accounts, concurrent multi-device (not WhatsApp identity)

**Status:** accepted

**Context:** WhatsApp ties identity to a phone number and a primary device (companions are linked, not independent logins). This product should work like Facebook Messenger: sign in with credentials on any device, several sessions at once, history from the server.

**Decision:** v1 identity is handle/email + password. Each login registers a `devices` row. Live events fan out to all of the user’s sockets. New devices hydrate via HTTP; there is no QR linking and no primary device. WhatsApp-style companion linking and Signal multi-device remain out of scope (see ADR-005).

**Consequences:** Outbox is per device; canonical messages live in Postgres. Receipts stay per `user_id`. Tokens are device-scoped so “log out this laptop” does not sign out the phone.
