# 14 — Decisions (ADR)

## ADR-001: Socket.IO first, Phoenix later

**Status:** accepted

**Context:** Need realtime quickly; team/JS ecosystem; migration planned.

**Decision:** Phase 1 gateways in Node + Socket.IO + Redis adapter. Abstract `ChatTransport`. Phase 2 Phoenix Channels, same protocol and Postgres. Host framework: **NestJS** (see ADR-011).

**Consequences:** Sticky LB; adapter semantics; discipline on isolation. Phoenix is not a rewrite.

## ADR-002: PostgreSQL as message source of truth

**Status:** accepted

**Context:** LSM stores scale ingest; they add ops cost.

**Decision:** Postgres + B-Tree `(chat_id, seq)` until measured pain. Redis not allowed to allocate `seq` or store canonical bodies.

## ADR-003: Client-generated idempotencyKey

**Status:** accepted

**Context:** At-least-once WS/HTTP.

**Decision:** UUIDv7 at outbox insert; JSON field `idempotencyKey`; unique SQL `(sender_id, idempotency_key)`; first write wins; `duplicate` flag on ack.

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

**Decision:** Field-stable **camelCase** JSON (TypeScript, Nest, Next, Socket.IO payloads, HTTP bodies and query params). Event names stay dotted (`message.send`) across Socket.IO and Phoenix. Postgres and SQLite **columns** stay snake_case; map at the ORM. Protobuf later is the same fields, not a different model.

**Consequences:** `idempotencyKey` on the wire; `idempotency_key` in SQL. JWT custom claim `deviceId` (`sub` unchanged). No snake_case in application types.

## ADR-008: Credential accounts, concurrent multi-device (not WhatsApp identity)

**Status:** accepted

**Context:** WhatsApp ties identity to a phone number and a primary device (companions are linked, not independent logins). This product should work like Facebook Messenger: sign in with credentials on any device, several sessions at once, history from the server.

**Decision:** v1 identity is an account (`users` row), not a phone. Sign-in is handle/email + password **and** Google OAuth; Keycloak (OIDC) is the same IdP seam later (ADR-012). Each login registers a `devices` row. Live events fan out to all of the user’s sockets. New devices hydrate via HTTP; there is no QR linking and no primary device. WhatsApp-style companion linking and Signal multi-device remain out of scope (see ADR-005).

**Consequences:** Outbox is per device; canonical messages live in Postgres. Receipts stay per `userId`. Tokens are device-scoped so “log out this laptop” does not sign out the phone. Google/Keycloak do not replace devices or the access JWT.

## ADR-009: Direct-to-store media; TUS for large files and unstable networks

**Status:** accepted

**Context:** Media is after v1 text. Chat apps fail on flaky mobile if a large PUT restarts from zero. Putting bytes on the gateway couples file transfer to Socket.IO/Phoenix and blows connection budgets.

**Decision:** Upload out of band. Small files on a stable link use presigned PUT to object storage. **TUS** is the upload protocol when the network is extremely unstable or the file is large (video / long voice). TUS terminates at a dedicated upload service that writes to S3 (or equivalent); then CDN + virus scan. `message.send` only references a `ready` `mediaId`. Same `idempotencyKey` / outbox as text.

**Consequences:** Extra upload service and client TUS libraries. Images stay simple PUT. Realtime protocol unchanged across Phase 1 → 2. See [10 — Resilience and scale](./10-resilience-and-scale.md).

## ADR-010: OTel + Collector + Tempo + Loki + Prometheus + Grafana

**Status:** accepted

**Context:** Need to learn production observability, not paste a cloud APM. Traces, logs, and metrics must correlate on `requestId` (W3C trace id). Direct app exporters hide the collector boundary. Zipkin and Grafana Alloy are coherent but less portable / less Grafana-native for traces than Tempo + OTel Collector.

**Decision:** Instrument with **OpenTelemetry** (OTLP only). **OpenTelemetry Collector** is the router. Backends: **Grafana Tempo** (traces), **Loki** (logs), **Prometheus** (metrics), **Grafana** (UI). Local Compose first. No Zipkin/Jaeger until Tempo in Grafana is fluent. No Alloy until the Collector config is fluent. Do not run Collector and Alloy together at the start.

**Consequences:** Extra Compose services; 100% sample in dev. Same span names on Phoenix. No message `body` in any signal. See [12 — Observability](./12-observability.md).

## ADR-011: NestJS as Phase 1 gateway host

**Status:** accepted

**Context:** Fastify is thinner and was the first sketch. Organization (modules, DI, guards) matters more for this repo than a few hundred lines of glue. NestJS is TypeScript, so `packages/protocol` stays shared with Next.js. FastAPI would split languages. Nest is extra machinery only if domain leaks into `@WebSocketGateway`.

**Decision:** `apps/gateway` is **NestJS + Socket.IO** (`@nestjs/platform-socket.io`). Nest is the host: HTTP controllers, WS gateways, auth guards. **Domain lives in injectable providers** (`ChatService`, receipts). Redis adapter attaches to the underlying Socket.IO `Server`, not a Nest microservice or CQRS bus. Phoenix remains the Phase 2 engine (ADR-001).

**Consequences:**

- Controllers and gateways parse, authorize, call a service, ack/return. No persist or seq logic in decorators.
- Do not use Nest microservices / event-emitter for chat fanout. Fanout is `@socket.io/redis-adapter` + Postgres.
- Tests target `ChatService` (and HTTP e2e), not only gateway classes.
- Phoenix cutover still drops Nest; the client `ChatTransport` is unchanged.

See [08 — Phase 1 Socket.IO](./08-phase-1-socketio.md).

## ADR-012: External IdPs (Google now, Keycloak later) are not the session

**Status:** accepted

**Context:** “Continue with Google” is expected. Keycloak (or any OIDC) may replace or sit beside Google later. IdP tokens expire on the IdP’s clock, have no `deviceId`, and cannot revoke one of our devices. NextAuth / Clerk as the source of truth would put sessions in Next while Socket.IO lives on Nest. If Keycloak issued the socket JWT, we would need custom mappers, Keycloak logout-per-device, and a JWKS rewrite of every guard.

**Decision:** Nest owns the **app session**. Any external login is: OIDC/OAuth **authorization code on the gateway** → upsert `users` + `identities(provider, subject)` → issue the **same** device-scoped access JWT + hashed refresh as password login. Socket handshake is **only** our JWT (`handshake.auth.token`). Next.js redirects to Nest; it does not own identity.

v1 providers: `password`, `google`. Later: `keycloak` (same `OidcLogin` path, new config). Do **not** validate Google or Keycloak access tokens on HTTP or Socket.IO. Do **not** put Keycloak in front of the chat protocol.

**Consequences:**

- `AuthService.issueSession(userId, device)` is the only place that mints JWTs. Password, Google, and Keycloak all end there.
- `identities` is `(provider, subject)` unique. Google `sub` and Keycloak `sub` are just rows. No `google_sub` column.
- `password_hash` stays on `users` (nullable). A user is valid with a password and/or at least one identity.
- Same verified email: **link** to the existing `users` row. Do not create a second account.
- New OIDC users still need a unique `handle` (prompt once, or generate from email and allow edit).
- Adding Keycloak: realm + client in Keycloak, env (`KEYCLOAK_ISSUER`, client id/secret), register provider `keycloak`, `GET /auth/oidc/keycloak`. No ChatService or transport changes.
- Logout / revoke device is unchanged (our refresh rows). Revoking the IdP account does not instantly drop our sockets — acceptable for v1.

See [11 — Security](./11-security.md), [07 — Data model](./07-data-model.md).
