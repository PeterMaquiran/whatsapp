# Messenger-style Chat — Architecture Docs

This folder is the design source of truth for Phase 1 (Next.js + NestJS + Socket.IO) and a planned Phase 2 migration to Elixir + Phoenix Channels.

**Product identity:** credential accounts (handle/email + password) that can be logged in on **multiple devices at once**, like Facebook Messenger. It is **not** WhatsApp (phone number + primary device + QR-linked companions). We still copy WhatsApp-like send/receipt/offline UX.

**Invariant:** UI and local offline logic never talk to Socket.IO or Phoenix directly. They talk to an abstract `ChatClient` / `Transport` plus a local outbox. That is what makes the later swap cheap.

| Doc | What to analyze |
| --- | --- |
| [01 Overview](./01-overview.md) | Identity (credentials + multi-device), product scope, delivery guarantees |
| [02 System architecture](./02-system-architecture.md) | Layered blueprint, process topology, request/event paths |
| [03 Transport abstraction](./03-transport-abstraction.md) | Interface, lifecycle, capability flags, swap rules |
| [04 Offline outbox](./04-offline-outbox.md) | SQLite/IDB schema, send pipeline, retry, conflict with server ids |
| [05 Idempotency](./05-idempotency.md) | `idempotency_key` contract, store, races, replay |
| [06 Message protocol](./06-message-protocol.md) | Events, envelopes, receipts, typing, presence |
| [07 Data model](./07-data-model.md) | PostgreSQL schema, Redis keys, indexes (B-Tree vs LSM) |
| [08 Phase 1 Socket.IO](./08-phase-1-socketio.md) | NestJS gateways, Redis adapter, sticky sessions, Docker |
| [09 Phase 2 Phoenix](./09-phase-2-phoenix.md) | Channel mapping, dual-run, cutover |
| [10 Resilience and scale](./10-resilience-and-scale.md) | Circuit breaker, rate limit, Kafka/NATS later, TUS media |
| [11 Security](./11-security.md) | Auth, TLS, E2E later, abuse |
| [12 Observability](./12-observability.md) | OTel Collector, Tempo, Loki, Prometheus, Grafana |
| [13 Learning roadmap](./13-learning-roadmap.md) | Fundamentals → distributed → scale → AI, mapped to this repo |
| [14 Decisions](./14-decisions.md) | ADRs: Socket.IO first, NestJS gateway host, Postgres first, credential multi-device, no E2E in v1, TUS media, OTel Collector + Tempo/Loki/Prometheus |
| [15 Sprints](./15-sprints.md) | GitHub milestones and issues: Next.js + NestJS gateway monorepo, Sprint 1 → 10, backlog |

Read order for a first pass: **01 → 02 → 05 → 06 → 08 → 09**. To build: **15**.
