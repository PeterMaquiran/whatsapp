# 11 — Security

## Transport

- WSS only in staging/prod (TLS at Nginx/ALB).
- HTTP APIs HTTPS.
- HSTS at the edge.

## Auth (credential accounts, multi-device)

Identity is **not** a WhatsApp phone / primary-device pair. It is an account the user signs into with credentials on as many devices as they want (Messenger-style).

- Register + login: handle/email + password (Argon2id). No phone required in v1.
- **External IdP** (Google in v1; Keycloak later): OIDC/OAuth authorization code **on Nest**. Upsert `users` + `identities(provider, subject)`, then `AuthService.issueSession` — the **same** access JWT + device refresh as password login. Do not put Google or Keycloak tokens on HTTP APIs or Socket.IO. Next.js is not the IdP (no NextAuth as source of truth).
- Short-lived access JWT (e.g. 15 min) claims: `sub` = user id, `deviceId`. Refresh token rotated, hashed in DB, **scoped to that device**. Refresh cookie: `HttpOnly` + `Secure` + `SameSite=Lax` when web and gateway share an origin (Nginx).
- Socket handshake: **our** access JWT only (`handshake.auth.token`). Join `user:{userId}` so all of that user’s devices get live events.
- `deviceId` in the token must exist, belong to `sub`, and not be revoked. Reject mismatches. A **second** device with its own id is valid, not a conflict.
- Logout / “log out this device” revokes that device’s refresh tokens. “Log out everywhere” revokes all devices.
- Password change: revoke all refresh tokens; other devices must log in again.

Do not implement “this login kicks the previous phone” unless the user explicitly chooses a single-session policy later. Default is concurrent sessions.

## Authorization

Every `message.send` / history GET:

- Membership check `chat_members`.
- Do not trust client `senderId`; take from token.

## Abuse

- Rate limits (see resilience doc).
- Body size cap.
- Ban table (later).

## Data

- Postgres encryption at rest (cloud default).
- Local Adminer and Redis Insight: **localhost only** (ADR-013). Not in prod.
- Logs / traces / metrics: never include message `body` (hash/id only). `userId` ok as a span/log field, **not** as a Loki label. Tokens redacted. Telemetry stack: [12 — Observability](./12-observability.md).

## E2E encryption (not v1)

WhatsApp-style Signal protocol is a **separate layer** above the transport:

- Server stores ciphertext; `body` becomes opaque blobs.
- Sealed sender, identity keys, WhatsApp-style multi-device ratchet — huge. Credential multi-device (this product) does not require that layer.
- Architecture still: outbox + idempotency (keys wrap ciphertext).

Do not pretend TLS is E2E. Docs and UI should not say “encrypted” until Signal (or MLS) exists.

## Multi-tenant / rooms

Join room only after authz. Socket.IO: never let the client pass arbitrary room names without server check.
