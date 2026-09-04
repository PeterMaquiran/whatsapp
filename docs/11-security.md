# 11 — Security

## Transport

- WSS only in staging/prod (TLS at Nginx/ALB).
- HTTP APIs HTTPS.
- HSTS at the edge.

## Auth

- Short-lived access JWT (e.g. 15 min), refresh token rotated, hashed in DB.
- Socket handshake: access token only.
- `device_id` bound to user; reject mismatched device.

## Authorization

Every `message.send` / history GET:

- Membership check `chat_members`.
- Do not trust client `sender_id`; take from token.

## Abuse

- Rate limits (see resilience doc).
- Body size cap.
- Ban table (later).

## Data

- Postgres encryption at rest (cloud default).
- Logs: never log message `body` in prod (hash/id only).
- PII in traces: `user_id` ok, tokens redacted.

## E2E encryption (not v1)

WhatsApp-style Signal protocol is a **separate layer** above the transport:

- Server stores ciphertext; `body` becomes opaque blobs.
- Sealed sender, identity keys, multi-device — huge.
- Architecture still: outbox + idempotency (keys wrap ciphertext).

Do not pretend TLS is E2E. Docs and UI should not say “encrypted” until Signal (or MLS) exists.

## Multi-tenant / rooms

Join room only after authz. Socket.IO: never let the client pass arbitrary room names without server check.
