# 08 — Phase 1: Node.js + Socket.IO

## Goals

- Two (or more) gateway processes, one Redis, one Postgres.
- Sticky sessions at the load balancer (Socket.IO HTTP long-polling fallback).
- All live fanout via `@socket.io/redis-adapter`.
- Domain logic in `ChatService`, not inside `socket.on` callbacks.

## Process topology

```
[Clients (WebSockets / WSS)]
                 │
                 ▼
   [Nginx / AWS Application Load Balancer]
     TLS offload
     sticky: ip or cookie (Socket.IO)
                 │
        ┌────────┴────────┐
        ▼                 ▼
[Node Gateway 1]   [Node Gateway 2]
 Fastify
 Socket.IO (WS adapter)
 Redis adapter client
 ChatService → Postgres
        │                 │
        └────────┬────────┘
                 │ @socket.io/redis-adapter
                 ▼
            [Redis]
                 │
                 ▼
          [PostgreSQL]
```

## Why sticky sessions

Socket.IO may start with HTTP long-polling, then upgrade. Without stickiness, poll and upgrade hit different nodes and the handshake breaks.

- **WebSocket-only** (`transports: ["websocket"]`) reduces stickiness need but still use stickiness in Phase 1.
- Phoenix Channels (Phase 2) also prefer stickiness for long-lived sockets; fanout is separate (PubSub).

Nginx sketch:

```nginx
upstream chat_gw {
  ip_hash;  # or sticky cookie
  server gateway1:3000;
  server gateway2:3000;
}

server {
  listen 443 ssl;
  location /socket.io/ {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Authorization $http_authorization;
    proxy_pass http://chat_gw;
  }
  location / {
    proxy_pass http://chat_gw;
  }
}
```

## Socket.IO server

- Auth in `io.use`: JWT from `handshake.auth.token`.
- Rooms: `user:{userId}` (all devices), `chat:{chatId}` (members currently joined), `device:{deviceId}`.
- Join `chat:{id}` only after membership check.
- `emit` to a chat: `io.to(`chat:${chatId}`).emit("message.created", payload)` — Redis adapter fans out to other nodes.

## Redis adapter vs Redis Streams

| | Redis adapter | Redis Streams |
| --- | --- | --- |
| Job | Copy Socket.IO packets between nodes | Durable event log |
| Persist | No | Yes |
| Phase | 1 | 2+ with Kafka/NATS |

If a node is down, adapter messages are missed by that node’s sockets; those clients reconnect and HTTP-sync seq. Do not confuse adapter with a message queue.

## Docker Compose (local)

Services: `postgres`, `redis`, `gateway` (scale=2), `nginx`.

Gateway env: `REDIS_URL`, `DATABASE_URL`, `JWT_SECRET`, `PORT`.

Health: `/healthz` liveness, `/readyz` checks PG + Redis.

## Node internals

```
apps/gateway
  src/http/          # REST sync
  src/ws/socket.ts   # bind events → use cases
  src/app/           # ChatService, Receipts
  src/infra/postgres
  src/infra/redis
```

`socket.on("message.send")` only:

1. Parse + validate
2. `userId` from socket data
3. `await chatService.send(...)`
4. ack
5. if `!duplicate` then `io.to(chat).emit(...)`

## Auth on the socket

Handshake without a valid token: reject. Token refresh: client disconnects and reconnects with new token (simple). Later: `auth.refresh` event.

## Horizontal scale limits (know them)

Socket.IO + Redis adapter is **push to connected sockets**, not a global bus for offline users. Offline delivery = persist in Postgres + push notification (later) + sync on connect.

Hot chats: every member socket gets an emit. Fine for 1:1. Groups of 1k+ need smarter fanout (Phase 3).
