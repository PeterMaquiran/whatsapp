# 13 — Learning roadmap (tied to this product)

Use the chat app as the lab, not isolated tutorials.

## Phase 1 — Deep fundamentals

| Topic | How it shows up here |
| --- | --- |
| TCP vs UDP | WS is TCP; calls later are UDP/QUIC |
| WebSockets | Gateway live plane; know ping/pong, backpressure |
| HTTP/2 | History API, muxed fetches; not a replacement for WS fanout |
| Protobuf | Second encoding of **the same** protocol (doc 06) |
| B-Trees vs LSM | Postgres B-Tree on `(chat_id, seq)` vs future Scylla message log |

**Exercises in-repo:** capture a WS frame; explain why seq is not `created_at`; run `EXPLAIN ANALYZE` on history query.

## Phase 2 — Distributed messaging & resilience

| Topic | How it shows up here |
| --- | --- |
| Redis pub/sub | Socket.IO adapter |
| Redis Streams / NATS / Kafka | Server outbox fanout |
| Circuit breaker | PG pool |
| Idempotency | Doc 05 — core feature |
| Rate limiting | Redis counters |

**Exercises:** kill Redis, confirm persist still works and sync repairs live; double-submit send; open breaker with a fault proxy.

## Phase 3 — High-scale system design

| Topic | How it shows up here |
| --- | --- |
| Messaging systems | This architecture |
| Feeds | `inbox` table |
| Rate limiters | Per-user, per-IP |
| OpenTelemetry | Doc 12 end-to-end Node → PG → Redis |

**Exercises:** two-node docker; chaos kill one gateway; load-test connections (be honest about Socket.IO vs Phoenix numbers).

## Phase 4 — AI & future-proof infra

| Topic | How it shows up here |
| --- | --- |
| RAG / vector DB | Optional: “search my chats” locally (client embeddings) or server-side later — **after** privacy policy. Do not dump bodies to a third-party LLM by default. |
| Edge / CDN | Media, not WS. WS stays regional. |
| Cloud-native | Compose → K8s, HPA on `ws_connected` and CPU |

## Suggested build order (engineering, not learning)

1. Postgres schema + HTTP auth + 1:1 chat create  
2. `ChatService.send` with idempotency (HTTP first — easier tests)  
3. Local outbox + fake transport  
4. Socket.IO transport + ack  
5. Redis adapter + 2 gateways  
6. Receipts + seq gap fill  
7. Nginx sticky + TLS  
8. OTel  
9. Phoenix transport spike behind a flag  
10. Media, groups, E2E, Kafka — only when 1–8 are boringly solid  
