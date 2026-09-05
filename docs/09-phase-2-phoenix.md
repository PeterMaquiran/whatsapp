# 09 — Phase 2: Elixir + Phoenix Channels

## Why Phoenix later

BEAM: cheap processes, preemptive scheduling, excellent WS density. Phoenix PubSub + Channels map cleanly to chat rooms. This is a **runtime swap**, not a product rewrite, if Phase 1 respected the transport interface.

## Mapping

| Socket.IO | Phoenix |
| --- | --- |
| namespace `/chat` | `UserSocket` |
| room `chat:{id}` | topic `chat:{id}` |
| `emit("message.send")` | `push("message:send")` or `"message.send"` |
| ack callback | `Phoenix.Channel` reply `{:ok, ack}` |
| `@socket.io/redis-adapter` | `Phoenix.PubSub` (Redis/PG later) |
| `io.use` JWT | `UserSocket.connect/3` |

Keep **JSON event names identical** (`message.send`) so `PhoenixTransport` is a thin encoder.

## PhoenixTransport.ts (client)

Uses `phoenix` JS client (or Swift/Kotlin equivalents):

- `socket.connect()` with `{ token, device_id }` after credential login (same multi-device model as Phase 1)
- `channel = socket.channel("chat:" + chatId)`
- `channel.push("message.send", payload).receive("ok", ack)`

`ChatClient` unchanged.

## Dual-run (recommended cutover)

```
                 ┌─────────────┐
  clients ───────│  LB / Nginx │
                 └──────┬──────┘
            /v1/ws      │      /v2/ws
                 ┌──────┴──────┐
                 ▼             ▼
           Node gateways   Phoenix nodes
                 │             │
                 └──────┬──────┘
                        ▼
              same PostgreSQL
              PubSub bridge (Redis)
```

1. Phoenix writes/reads the **same** `messages` table and idempotency rules.
2. A bridge: Node publishes to Redis `chat:{id}`; Phoenix subscribes (and reverse) so mixed clients work.
3. Move traffic % by client flag `transport=phoenix`.
4. Decommission Node when echo tests pass (duplicate keys, two-node fanout, gap fill).

Do **not** run two independent seq allocators. Seq bump stays in Postgres (`UPDATE chats SET current_seq`).

## What you rewrite

- Gateway process and Channel modules
- Presence: `Phoenix.Presence` (still Redis-backed in cluster)

## What you do not rewrite

- Outbox, UI, `ChatTransport` consumers
- Postgres schema
- HTTP history API (can stay Node or move to Phoenix; contract unchanged)
- Idempotency semantics

## BEAM-specific wins (later)

- Per-chat `GenServer` for hot rooms (optional; do not start here)
- Better tail latency under huge connection counts

## Risk

If Phase 1 leaks Socket.IO acks, rooms, or engine.io packet types into domain code, migration explodes. Code review gate: **no `socket.io` imports outside `/infra/socketio`.**
