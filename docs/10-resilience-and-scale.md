# 10 — Resilience and scale

Mapped to your learning phases; **implement incrementally**. Phase 1 ships the first column only.

## Phase 1 (must have)

| Pattern | Implementation |
| --- | --- |
| Idempotency | `(sender_id, idempotency_key)` unique; client outbox |
| At-least-once + merge | Retry same key; UI merge |
| Timeouts | Transport ack timeout; in_flight recovery |
| Rate limit | Redis token bucket per `user_id` on send (e.g. 20/10s) |
| Backpressure | Drop typing first; never drop persist |
| Load shedding | Reject new sockets with `UNAVAILABLE` if PG pool exhausted |
| Sticky LB + 2 nodes | Prove Redis adapter |
| Gap fill | HTTP `after_seq` |

### Circuit breaker (gateway → Postgres)

If PG errors exceed N in a window: open breaker, fail sends with `UNAVAILABLE`, still accept connections for sync when half-open. Do not retry INSERT in a tight loop on a dead primary.

### Rate limiting

```
INCR rl:send:{userId}
EXPIRE 10
if > 20 → ack RATE_LIMITED (outbox backoff)
```

Separate limits: login, typing (cheaper, higher cap).

## Phase 2 (distributed messaging)

When Redis pub/sub misses (node crash) become painful:

```
ChatService.commit
  → INSERT server_outbox
  → commit
Publisher → NATS / Kafka / Redis Streams
  → all gateways + push workers
```

- **Redis Streams:** consumer groups, cheap, ops-simple; retention limited.
- **NATS JetStream:** good for fanout + request/reply.
- **Kafka:** when you need long retention, many consumer types (search index, ML, analytics).

Client protocol does not change. Only the server’s post-commit fanout changes.

## Phase 3 (high-scale design)

| Concern | Direction |
| --- | --- |
| Connection density | Phoenix or specialized gateways |
| Message store | Partition by `chat_id` (Citus/Scylla) |
| Unread counts | Redis / counter table, not `COUNT(*)` |
| Hot group fanout | Hybrid: push to online set in Redis, not O(members) on PG |
| Multi-region | Regional gateways, single writer for seq or CRDT (hard — avoid until needed) |
| Media | Direct-to-object-store; **TUS** for large files / unstable networks; CDN; virus scan |

### Media uploads (TUS when the network or size is hostile)

Bytes **never** go through Socket.IO / Phoenix or `message.send`. The chat path stays a small JSON envelope (`media_id`, type, size, dimensions, thumbnail). Recipients fetch via CDN.

**Default (small images / short clips on a stable link):** authenticated **presigned PUT** (or POST) straight to S3/R2/GCS. Retry the whole object. Idempotency is the object key (`user_id` + `upload_id`).

**TUS (tus.io) is required when either is true:**

- **Extremely unstable network** — mobile, high packet loss, frequent disconnects, long stalls. A single PUT that dies at 80% would restart from byte 0; TUS resumes at the last acknowledged offset (`PATCH` + `Upload-Offset`).
- **Large files** — long video / voice where a full retry is expensive (roughly tens of MB and up; treat ~20–50MB as the split, measure later).

TUS sits **in front of** object storage (`tusd` or equivalent with an S3 store). Clients (web / iOS / Android) share one resume protocol. Pause/resume is first-class UX, same as the text outbox: local row stays `pending` until the upload is `complete` and the chat send acks.

```
Client (tus-js / native)
  → TUS (Creation + PATCH offsets, checksum)
  → object store
  → scan / transcode (async)
  → media_id = ready
ChatClient.sendMessage({ type, media_id, … })  // existing idempotency_key
```

Do **not** use TUS for every thumbnail. Do **not** multiplex file bytes on the realtime socket. Incomplete TUS uploads expire; the chat message is not inserted until `ready`.

### Feed / inbox

Chat list is a **materialized inbox**:

```sql
CREATE TABLE inbox (
  user_id UUID,
  chat_id UUID,
  last_seq BIGINT,
  last_preview TEXT,
  last_at TIMESTAMPTZ,
  unread INTEGER,
  PRIMARY KEY (user_id, chat_id)
);
```

Update in the send transaction. List query is then `ORDER BY last_at DESC`.

## Failure modes cheat sheet

| Failure | User impact | Mitigation |
| --- | --- | --- |
| Redis down | Cross-node live fail; same-node may work | Still persist PG; clients HTTP sync; alerts |
| One gateway down | Those sockets drop | Client reconnect; sticky to living node |
| PG primary down | Sends fail | Breaker; do not lie with local-only forever without “pending” |
| Duplicate send | None if keys work | Unique index |
| Split brain seq | Impossible if seq in PG | Never allocate seq in Redis |
| Poison message | One chat stuck | Terminal validation; skip + metric |
| Upload drop (unstable / large) | Media stuck “sending” | TUS resume from offset; expire incomplete; send only after `ready` |

## Idempotency vs exactly-once

Network is at-least-once. **Exactly-once effect** = idempotent handler + unique store. This is the whole trick; Kafka “exactly once” is not required in Phase 1.
